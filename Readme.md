# OpenStreetMap Project
## Data Wrangling with MongoDB

---------------------
**Map Area**: Reading, England, United Kingdom

**Mapzen download link**:
https://s3.amazonaws.com/metro-extracts.mapzen.com/reading_england.osm.bz2

## 1. Problems Encountered in the Map

-	Tags with dots (e.g. addr.source:housenumber)
-	Streets without a street type (e.g. Brerewood, Marefield, Crossways)
-	Street names that start with “The” followed by another word (e.g. “The Chase”)
-	Street names with commas (E.g. “Riverside, The Oracle”)
-	Street types that were difficult to anticipate (e.g. “Approach”)

### Tags with dots
Tags.py was used to get a count of the tags that are lower, lower with a colon, problematic and other.
```
{'lower': 135330, 'lower_colon': 130250, 'other': 562, 'problemchars': 1}
```

I have printed the problematic tag and found that the problem is the presence of a dot instead of a colon:
```
addr.source:housenumber
```
This should have probably been  “source:housenumber” but being the only occurrence of the source tag for housenumber it can be ignored.
This turned out to be a problem also when importing the data to MongoDB  so the code had to deal with that by replacing the dot with a colon if encountered in the line.
```python
if "addr.source" in line:
    line = line.replace ("addr.source", "addr:source")
```
### Street suffix that were difficult to anticipate
To get a full list I modified audit.py and added a try/except statement so that if the street type is not found in the try part it will be printed in the except.
```python
for st_type, ways in st_types.iteritems():
        for name in ways:
            try:
                better_name = update_name(name, mapping)
                print name, "=>", better_name
            # This should not print anything if all "expected" have been added
            except:
                print name
```
Then I have updated the “expected” list and found the issues below.

### Street names without a type
When the street name is only one word their type is absent so I had to modify the audit_street_type function in audit.py to also accept one word streets. There are 10 cases and I can't be sure about all of them but at least some (e.g. Queensway, Marefield) are genuine street names so I assumed they are all fine.

### Streets starting with “The”
There are 7 and are genuine street names.

### Street names with commas
There are 7 cases and they seem to be due to the fact that the user or the bot tried to put more information than what is supposed to go under the "addr:street" tag. For example "House of Fraser, The Oracle Shopping Centre" and "Luscinia View, Napier Road". I couldn't find a way to deal with the issue as the street part of the name is not consistently on the left or right of the comma and in some cases is entirely absent.

In the end, the final part of the audit_street_type function looked like this:

```python
# Exit the function if:
        if not words_count > 1:
            return
        if first_word == "The":
            return
        if name_has_comma:
            return
        if street_type in expected:
            return
        
        # Add if none of the above exited the function
        street_types[street_type].add(street_name)
```

### Conclusion on the problems
The final audit.py script in the end only found two street names with an inconsisten name type:

>	Church Rd => Church Road

>	Gun street => Gun Street

The data is overall very clean for the street names. Here is in short the list of real issues that i have found:

* 1 case of abbreviation
* 1 case of "Street" spelled without a capital letter
* 7 street names with a comma

## 2. Data Overview

### Size of files:

-	reading_england.osm: 73MB
-	reading_england.osm.json: 82MB

Once the json data was imported to a collection under localhost I could run some queries.

### Number of documents
```python
print city_openStreetMap.count()
359072
```

### Number of nodes
```python
print city_openStreetMap.find({"type":"node"}).count()
293662
```

### Number of ways
```python
print city_openStreetMap.find({"type":"way"}).count()
65406
```

### Number of unique users
```python
print len(city_openStreetMap.find({}).distinct("created.user"))
454
```

### Top contributor
```python
pipeline = [{"$group":{"_id":"$created.user", "count":{"$sum":1}}},
            {"$sort": {"count" : -1}},{"$limit":1}]
print aggregate(pipeline)

[{u'count': 277226, u'_id': u'Eriks Zelenka'}]
```

### Users with only one contribution
```python
pipeline = [{"$group":{"_id":"$created.user", "count":{"$sum":1}}},
            {"$group":{"_id":"$count", "num_users":{"$sum":1}}},
            {"$sort":{"_id":1}}, {"$limit":1}]
print aggregate(pipeline)

[{u'num_users': 90, u'_id': 1}]
```

### Top 10 amenity types
```python
pipeline = [{"$match": {"amenity": {"$exists": 1} } },
            {"$group": {"_id":"$amenity", "count": {"$sum":1} } },
            {"$sort": {"count":-1} },{"$limit": 10}]
print pprint.pprint(aggregate(pipeline))

[{u'_id': u'parking', u'count': 323},
 {u'_id': u'post_box', u'count': 223},
 {u'_id': u'bench', u'count': 185},
 {u'_id': u'bicycle_parking', u'count': 136},
 {u'_id': u'pub', u'count': 118},
 {u'_id': u'waste_basket', u'count': 111},
 {u'_id': u'place_of_worship', u'count': 98},
 {u'_id': u'fast_food', u'count': 98},
 {u'_id': u'school', u'count': 97},
 {u'_id': u'telephone', u'count': 73}]
```

## 3. Additional Ideas
Something I’ve noticed is that some entries are very old, and the oldest ones are from August 2006. This is not a problem for streets, parks etc. but amenities like shops close and open frequently so very old entries have a chance of being incorrect now.

Just as an example, the following query returns the two documents referring to amenities with the oldest timestamps:

```python
pipeline = [{"$match": {"amenity": {"$exists": 1} } },
            {"$match": {"created.timestamp": {"$exists": 1} } },
            {"$sort": {"created.timestamp":1} },{"$limit": 2}]            
  
print pprint.pprint(aggregate(pipeline))
[{u'_id': ObjectId('56caf51125461c1374a09d9f'),
  u'amenity': u'post_box',
  u'created': {u'changeset': u'93630',
               u'timestamp': u'2006-08-27T00:34:21Z',
               u'uid': u'1295',
               u'user': u'robert',
               u'version': u'1'},
  u'created_by': u'JOSM',
  u'id': u'14096079',
  u'pos': [51.4577249, -0.9359429],
  u'type': u'node'},
 {u'_id': ObjectId('56caf51125461c1374a09da0'),
  u'amenity': u'post_box',
  u'created': {u'changeset': u'93630',
               u'timestamp': u'2006-08-27T00:34:22Z',
               u'uid': u'1295',
               u'user': u'robert',
               u'version': u'1'},
  u'created_by': u'JOSM',
  u'id': u'14096080',
  u'pos': [51.4573496, -0.9298445],
  u'type': u'node'}]
```

This could be a way of establishing the priority of entries to be reviewed.
It will obviously be difficult to go through each one of them but one way to approach this would be to implement a system to periodically ask the user who originally entered the data to update it since they are most likely to know if an update is needed.

## Conclusion

The source data downloaded from Mapzen was well formatted and only one error had to be fixed to make it possible for the json file to be loaded into mongodb. The script to audit the street names reveald a few minor issues but overall the format was very consistent.

A problem that I found is that there are about 650 documents referring to amenities that are at least five years old, and a portion of them must be in need of an update by now.



