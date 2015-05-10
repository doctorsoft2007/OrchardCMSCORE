## Data Layer Design

Entity Framework as the initial ORM (This may be swapped out later) because it is CoreCLR compliant.

The data layer is to be re-factored to use a Map Reduce structure.

Notes:

Store

Content Manager -> (Content Item)
    Content Item Store -> (Content Item)
        -> Content Storage Provider (Content Item Record)
            -> Content Document Store (Document Record)
            -> Content Index Provider (Document Record)
        
        -> Content Storage Provider (Content Item Version Record)
            -> Content Document Store (Document Record)
            -> Content Index Provider (Document Record)
            
            
-> Content Document Store (Document Record) (Type: EF Content Document Store)
    -> Data Context Add(Document Record)
    
-> Content Index Provider (Document Record) (Type: EF Content Index Provider)
    -> Calls multiple IContentQueryExpressionHandler's to receive map and sort lambda
    -> Creates Document Index Record to store index : (TODO: Remove and create dynamic tables for indexes)
    

Get

Store(map, sort)
 -> setting says where individual content items are stored (for now everywhere)
 - store data as well in index
 
Get(map, sort, reduce)
 -> search by map and sort then apply reduce
 -> if index not found, shell out to create index.

Pass a lambda in, and reduce.
    
system asks everyone 

distributed getmany....

1 index provider, and multiple data store

Note:
ContentItemRecord and ContentItemVersionRecord are both of type DocumentRecord

TODO: Async Story