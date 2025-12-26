Split the entire globe horizontally and vertically marking each side of split as 0 or 1 and keep splitting further to get to granular area. Doing this you end up with two separate numbers one for horizontal split, one for vertical- both looking like `1001011100`. 
### Spatial Locality

The points will be as close as their shared prefix length. One option is we keep going from left to right for both the numbers to determine the proximity or we can interleave both the numbers and resulting single number would also achieve the same purpose.

Instead of keeping a large string of  `0s`and `1s`, it can be encoded into Base32 (thereby absorbing 5 bits into one character) and hence reduce the size of string for storing and communication e.g. `u4pruydqq`

### Efficient Proximity Searches

We use Redis [[Sorted Set]] to convert a complex two dimension proximity problem into one dimensional range lookup.  In order to do this we perform following steps
- First do quick traversal based on the lat and long and build the string by doing repeated chain of split e.g. for latitude [-90, +90] and where it lies and then making it more granular. 
- Keep doing it 32 times for Lat and 32 times for Long- interleave them both, it will preserve the order and give you a 64 bit integer.
- Use this 64 bit integer as the score value for redis entry and then use `ZSET` to pull nearby records in `log N + M` time. 
- However one minor thing is that Redis can't store longer than 52 bits for score(or in general?)so it uses an encoding called coordinate-to-integer-geohash-encoding to store without loss. It effectively takes first 26 bits of both Lat and Long and it can give pretty high precision `<1m x <1m`area. We can do with much smaller - even 45 bits(9 chars of base32 encoded string)  can  cover `~5m X ~5m`
### Seam problem 

Two points which are very close to each other e.g. 1 meter apart could belong to different grids separated by the grid line. This would lead them to have different prefix- so we can't just query within the grid of one point's location to get closest points as there could be points closer which are in different grids.

This can be a problem when you're string prefix based searches but isn't a problem for Redis since it does a numerical range query. When you do run into this issue, you counteract it by looking up all 8 neighboring grids of current grid and applying post-filtering.

### Redis [[GeoSearch]]

1. Identify the optimal Geohash resolution:  Query for points-in-a-distance is typically`GEOSEARCH locations FROMLONLAT $lon $lat BYRADIUS 10 km`. In order to achieve this redis needs to settle on a granularity of origin/central grid (think zoom size in google map based on the input distance parameter). It tries to come up with the most granular (i.e. longest geohash) which can cover a significant portion lf the grid. Note that redis has internal table of `which length geohash covers how big of an area (e.g., a Geohash of length 6 corresponds to a cell size of about 610m x 610m)` 
2. Identify the 1 + 8 cells: Once the precision is chose, redis first identifies what is the geohash for the origin point itself. And to avoid Seam Problem, redis can also determine what the  neighboring 8 grids are. This setup of nine rectangular grids ensure that all points are covered.
3. Executing Range Queries tp get all the points of these cells: Redis can find out what the `min`and `max`score for each of the cell is. How? Because you can find the lowest point/score of a grid by setting 0 on the bits on the right side and highest by setting it to 1. After this, it will execute this query for each of the cells `ZRANGEBYSCORE key Min Score Max Score`
4. Post-filtering: At this point, redis has all points belong to these cells but many of those are not within required distance to redis does a post-filtering to drop points which arent in given distance based on their lat-long and a quick computation. 
### Precision

| **GeoHash Precision** | **Approx. Cell Size (Width x Height)** |
| --------------------- | -------------------------------------- |
| 5 Characters          | $4.89 \text{km} \times 4.89 \text{km}$ |
| 7 Characters          | $153 \text{m} \times 153 \text{m}$     |
| 9 Characters          | $4.77 \text{m} \times 4.77 \text{m}$   |