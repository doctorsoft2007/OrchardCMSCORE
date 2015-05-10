## Data Layer Design

Entity Framework as the initial ORM (This may be swapped out later) because it is CoreCLR compliant.

The data layer is to be re-factored to use a Map Reduce structure.

Notes:

### Store(map, sort)
-> setting says where individual content items are stored (for now everywhere)
- store data as well in index

### Get(map, sort, reduce)
-> search by map and sort then apply reduce
-> if index not found, shell out to create index.

Pass a lambda in, and reduce.
    
system asks everyone 

distributed getmany....

1 index provider, and multiple data store

Note:
ContentItemRecord and ContentItemVersionRecord are both of type DocumentRecord

TODO: Async Story