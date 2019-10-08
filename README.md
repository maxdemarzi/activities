# Activities
See activities in Neo4j

	CREATE CONSTRAINT ON (e:Email) assert e.address IS UNIQUE;
	CREATE CONSTRAINT ON (a:ActivityType) assert a.name IS UNIQUE;

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
	CREATE (e)-[:PERFORMED]->(a)


	
	MATCH (e:Email)-[r:PERFORMED]->(a)
	WITH e, a ORDER BY a.date ASC
	WITH e, COLLECT(a) AS activities, HEAD(COLLECT(a)) AS first
	   FOREACH (n IN RANGE(0, SIZE(activities)-2) |
	    	FOREACH (prev IN [activities[n]] |
	    		FOREACH (next IN [activities[n+1]] |
	    			CREATE (prev)-[:NEXT]->(next)
				)))
	CREATE (e)-[:NEXT]->(first)
	

