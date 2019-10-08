# Activities
See activities in Neo4j


Create our Schema:

	CREATE CONSTRAINT ON (e:Email) assert e.address IS UNIQUE;
	CREATE CONSTRAINT ON (a:ActivityType) assert a.name IS UNIQUE;

Copy the activities.csv file included in this repository to your Neo4j "import" directory.

Import the data:

	USING PERIODIC COMMIT
	LOAD CSV WITH HEADERS FROM 'file:///activities.csv' AS row
	MERGE (e:Email {address:row.email});

	USING PERIODIC COMMIT
	LOAD CSV WITH HEADERS FROM 'file:///activities.csv' AS row
	MERGE (at:ActivityType {name:row.activity});

	USING PERIODIC COMMIT
	LOAD CSV WITH HEADERS FROM 'file:///activities.csv' AS row
	MATCH (e:Email {address:row.email})
	CREATE (a:Activity {date: date(row.date), type:row.activity})
	CREATE (e)-[:PERFORMED]->(a);


Link the activities in an ordered list:

	MATCH (e:Email)-[r:PERFORMED]->(a)
	WITH e, a ORDER BY a.date ASC
	WITH e, COLLECT(a) AS activities, HEAD(COLLECT(a)) AS first
	CREATE (e)-[:NEXT]->(first)
	FOREACH (n IN RANGE(0, SIZE(activities)-2) |
		FOREACH (prev IN [activities[n]] |
			FOREACH (next IN [activities[n+1]] |
	    		CREATE (prev)-[:NEXT]->(next)
			)));


See the full list of activities for one email address:

	MATCH p=(e:Email {address: "duchamp@hotmail.com"})-[:NEXT*]->(a)
	WHERE NOT (a)-[:NEXT]->()
	RETURN p

Get the distinct activity types for one email address


	MATCH p=(e:Email {address: "duchamp@hotmail.com"})-[:NEXT*]->(a)
	RETURN COLLECT(DISTINCT a.type)	

Get the count of activity types one to three steps from an opportunity:

	MATCH p=(a1:Activity)-[:NEXT*1..3]->(a2:Activity)
	WHERE a2.type = "opportunity"
	RETURN [x IN NODES(p) | x.type], COUNT(p)
	ORDER BY count(p) DESC

Compare the distinct activities of two email addresses:

	MATCH p=(e:Email {address: "duchamp@hotmail.com"})-[:NEXT*]->(a),
		p2=(e2:Email {address: "birddog@mac.com"})-[:NEXT*]->(a2)
	WITH COLLECT(DISTINCT a.type) AS e_activities, 
	     COLLECT(DISTINCT a2.type) AS e2_activities
	RETURN [x IN e_activities WHERE NOT(x IN e2_activities)] AS unique_e, 
	       [x IN e2_activities WHERE NOT(x IN e_activities)] AS unique_e2, 
	       [x IN e_activities WHERE (x IN e2_activities)] AS common

todo:
     
	// Jaccard similarity 
    CREATE (e)-[:SIMILAR_ACTIVITIES {score:xx.yy}]->(e2)
