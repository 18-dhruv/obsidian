 - ### Read through
 ![InkWriting](<Ink/Writing/2026.10.18 - 21.47pm.svg>) [Edit Writing](https://youtu.be/2arL1jh8ihA?type=inkWriting&aspectRatio=5.333)

This approach simplifies application logic  because the application does not need to handle the logic for fetching and updating the cache.

The cache itself handles both reading from the database and storing the requested data automatically. This minimizes unnecessary data in the cache and ensure that frequently accessed data is readily available 

But for cache misses, there is a potential delay while the cache queries the database and stores the data.
To prevent the cache from serving stale data, a time-to-live(T T L ) can be added to cache enteries


