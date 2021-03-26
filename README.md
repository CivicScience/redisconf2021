# Crawl

## "Hello World"
Given a family of hashes in Redis named `profile_<ID>` with the structure:
* key={"value", "time"} 
* field=the name of my field 
* returns value=the value the respondent with ID has for the field. 
  
I hacked up the following to get the distribution of counts of the cross
product of the possible values for fields A and B by doing:
```
bg = GearsBuilder('KeysReader')
bg.filter(lambda x: 'A' in x['value'] and 'B' in x['value'])
bg.groupby(lambda r : r['value']['A'] + ',' + r['value']['B'], lambda key,a,r: 1 + (a if a else 0))
bg.run("profile_*")
```

# Walk
At this point I have profiles and  inverted indexes based on sets. In particular I have
three classes of indexes:
* Everyone (e.g. contains all respondents, good for "not" boolean operations) = `all`
* The respondent has a non-null value for the field = `any_<field>`
* The respondent has a specific value for the field  = `value_<field>_<field value>`

These indexes and indexes we can dynamically compute from them on the fly
are equal in size to the answer in many cases, examples: `A is not null` or
`A = 4 && B = 3 ||  C in {3,5}`. There are others that are not equal in size,
but I know the index (or derived index) contains a subset of respondents that
do match. Thus:
```
if expression.inverted_index.cardinality_matches_expression:
  return RedisDatabase.SCARD(expression.inverted_index.name)
else:
  count = 0
  foreach respondent in RedisDatabase.SMEMBERS(expression.inverted_index.name):
    p = Profile.load(respondent, expression.columns) 
    if expression.matches(p): 
      count += 1
  return count    
```

# Run (and where we are now)
The loop over respondents was slow and hashes used a lot of memory. By switching to sorted
sets I was able to ensure all possible queries were able to be answered from the indexes
themselves. But then to reduce memory I was able to convert low cardinality columns back into sets.

Some snippets of what the current solution looks like for a count command, example:
`count from table where x>3`

## RedisGears trigger registration:

```
def doCommand(args):
    """Do command."""
    try:
        db = MyRedisProxy()
        triggerName = args[0]
        # the only reason for this argument is to route command to the right shard for writes
        # or to a random shard for reads
        randomStringForReadsOrProfileHashKeyForWrites = args[1]
        cql = " ".join(args[2:])
        return db.executeCQL(cql)
    except Exception as e:
        myLog(f"doCommand args[{args}] exception[{e}]")
        raise e
        
GearsBuilder(
    "CommandReader",
    desc="count randomString CQL",
).map(doCommand).aggregate(0, count_sum, count_sum).register(
    trigger="count", mode="async"
)
```
Where count_sum is:
```
def count_sum(a: int, r: int) -> int:
    return a + r
```
The overall effect is that we:
* registered a trigger named "count" 
* have our library return a partial result per shard (db.executeCQL)
* have gears aggregate the partial results into the final answer using aggregate

We support many other command types with CQL, this is just one example. As you might imagine other
commands have other post-doCommand chains of functions. 

But you might ask: what is happening inside of the library?

## Converting CQL to objects
We have Antlr grammar that looks something like this:
```
compare_expr: column=Identifier op=compare_op compareValue=Integer;
```
and then we override the generated visitor base class to create objects.

## Objects to answers
All expression objects have:
```
    @abstractmethod
    def get_inverted_index(self) -> InvertedIndex:
        """Gets the inverted index for this Expression.."""

```
For our example this method gets me an object that represents the idea of an inverted index
constrained by a range (x > 3). InvertedIndexes have methods like:
```
    @property
    @abstractmethod
    def index_type(self) -> str:
        """'set' or 'sorted_set'."""

    @property
    @abstractmethod
    def key_name(self) -> str:
        pass

    @abstractmethod
    def delete(self) -> None:
        """Deletes this index (only the transient ones of course)."""
        
    @abstractmethod
    def persist(self) -> None:
        """Persists this index into Redis if it's persistable (otherwise a no-op)."""
```

These methods allow me to easily write:
```
    @staticmethod
    def size(inverted_index: InvertedIndex) -> int:
```
which basically manages persisting the set (if needed) and doing a ZCARD vs SCARD.
