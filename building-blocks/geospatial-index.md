# Geospatial indexes

Location information consists of two dimensions. 
Traditional hash indexing on either on of the attributes does not uniquely represent the location information.
Geo-indexing encodes well-defined areas using strings.

An example encoding is via a quad tree. 
A planar surface can be divided into four quadrants, viz,  a , b, c and d.
A point in any one of the quadrants is indexed by the string representation of its quadrant.

The quadrant can be subdivided into four more quadrants. 
If a is the quadrant to be subdivided, the child quadrants of a would be aa, ab, ac and ad.
This division can go on until the required granularity. 
This can be specified as a quadrant being no larger than x m.

Illustration
1. First level quadrants: a, b, c, d
2. Second level quadrants w.r.t a: aa, ab, ac, ad
3. Third level quadrants w.r.t. ab: aba, abb, abc, abd
4. Fourth level quadrants w.r.t aad: aada, aadb, aadc, aadd
5. and so on

A portion of the geo index can be visualized as 
```json
{
  "babcdabcdda" : [list of coordinates],
  "babcdabcddb" : [list of coordinates],
  "babcdabcddc" : [list of coordinates],
  "babcdabcddd" : [list of coordinates],
  "babcdabdaaa" : [list of coordinates],
  "babcdabdaab" : [list of coordinates]
}
```
## Operations
### Insertion
Given a coordinate, the grid it is mapped to can be determined by binary search based approach.
1. Start with the outermost grids and determine to which grid the given location belongs
2. This can be done by comparing the x and y coordinates with the grid boundaries
3. Once the outer grid is identified, drill down into its child grids until the leaf grids are reached
4. Complexity is going to be log<sub>4</sub>(n). n is the total number of nodes in the quad tree

### Retrieval 
A retrieval is typically to answer the use case of 'what are the other coordinates within x meters of the given location'
1. Determine the grid for the given location
2. If 'x meters' within protrudes to neighboring grids, get all the locations in those grids along with this one