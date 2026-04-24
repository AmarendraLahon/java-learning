Q1. Where does equals() without hashCode() break?
In HashMap and HashSet. Equal objects may go to different buckets, causing duplicates or retrieval failures.

Q2. What happens with intern()?
intern() returns reference from String Pool, so both references point to the same object and == returns true.

Q3. Integer caching behavior?
Values between -128 to 127 are cached, so references are same; outside this range, new objects are created.

Q4. Same hashCode but equals false?
Both objects go into the same bucket. equals() differentiates them, so both are stored.

Q5. Best collection for LRU cache?
LinkedHashMap with accessOrder=true, as it maintains access order and allows efficient eviction.

Q6. Why is String immutable but StringBuilder is not?
String is immutable for security, thread safety, string pooling, and hashCode caching. StringBuilder is mutable for performance in scenarios with frequent modifications.

Q7. Why does HashMap allow one null key but multiple null values?
HashMap allows one null key because it is stored in a fixed bucket (index 0) and keys must be unique. Multiple null values are allowed since values do not affect hashing or uniqueness.

Q8. What happens if a mutable object is used as a key in HashMap?
If the object changes after insertion, its hashCode may change, causing it to be stored in the wrong bucket and making retrieval or removal fail.