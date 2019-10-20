# Activities
See activities in Neo4j


Create our Schema:

	CREATE CONSTRAINT ON (e:Email) assert e.address IS UNIQUE;
	CREATE CONSTRAINT ON (at:ActivityType) assert at.name IS UNIQUE;
	CREATE INDEX ON :Activity(type);

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


If you have a large dataset, it may be better to link the activities in batches. You can run this query until no more relationships are created:

	MATCH (e:Email) 
	WHERE SIZE((e)-[:NEXT]->()) = 0 AND SIZE((e)-[:PERFORMED]->()) > 0
	WITH e
	LIMIT 1000
	MATCH (e)-[r:PERFORMED]->(a)
	WITH e, a ORDER BY a.date ASC
	WITH e, COLLECT(a) AS activities, HEAD(COLLECT(a)) AS first
	CREATE (e)-[:NEXT]->(first)
	FOREACH (n IN RANGE(0, SIZE(activities)-2) |
		FOREACH (prev IN [activities[n]] |
			FOREACH (next IN [activities[n+1]] |
	    		CREATE (prev)-[:NEXT]->(next)
			)));


See the full list of activities for one email address:

	MATCH p=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
	WHERE NOT (a)-[:NEXT]->()
	RETURN p

Get the distinct activity types for one email address:

	MATCH p=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
	RETURN COLLECT(DISTINCT a.type)	

Get the count of activity types one to three steps from an opportunity:

	MATCH p=(a1:Activity)-[:NEXT*1..3]->(a2:Activity)
	WHERE a2.type = "opportunity"
	RETURN [x IN NODES(p) | x.type], COUNT(p)
	ORDER BY count(p) DESC

Compare the distinct activities of two email addresses:

	MATCH p=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a),
		p2=(e2:Email {address: "ghost5@msn.com"})-[:NEXT*]->(a2)
	WITH COLLECT(DISTINCT a.type) AS e_activities, 
	     COLLECT(DISTINCT a2.type) AS e2_activities
	RETURN [x IN e_activities WHERE NOT(x IN e2_activities)] AS unique_e, 
	       [x IN e2_activities WHERE NOT(x IN e_activities)] AS unique_e2, 
	       [x IN e_activities WHERE (x IN e2_activities)] AS common

Count the number of activities that followed each other by type:

	MATCH (a1:Activity)-[:NEXT]->(a2:Activity)
	WITH a1.type AS a1type, a2.type AS a2type, count(*) AS count
	MATCH (at1:ActivityType {name: a1type}), (at2:ActivityType {name: a2type})
	MERGE (at1)-[r:NEXT]->(at2)
	SET r.count = count;

Top random walk by counts:

	MATCH path = (at:ActivityType)-[:NEXT*..5]->()
	WHERE ALL (r IN relationships(path) WHERE r.count > 1)
	RETURN [at IN nodes(path)| at.name] AS actions, reduce(sum=0, r IN relationships(path) | sum + r.count) AS score
	ORDER BY score DESC
	LIMIT 1

What is the most likely next activity for this user to take based on their last 3 activities:

	PROFILE MATCH path=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
	WHERE SIZE((a)-[:NEXT]->()) = 0
	WITH [at IN nodes(path)| at.type][-3..] AS last
	MATCH (a1:Activity)-[:NEXT]->(a2)-[:NEXT]->(a3)-[:NEXT]->(a4)
	WHERE a1.type = last[0] AND a2.type = last[1] AND a3.type = last[2]
	RETURN a4.type, COUNT(*) AS count
	ORDER BY count DESC

What if we created custom relationship types for the next action using APOC?

	MATCH (a1:Activity)-[:NEXT]->(a2:Activity)
	CALL apoc.create.relationship(a1, "NEXT_" + toUpper(a2.type), {}, a2) YIELD rel
	RETURN COUNT(rel)

Then we could query like this, and check just the relationship type instead of the next node property:

	PROFILE MATCH path=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
	WHERE SIZE((a)-[:NEXT]->()) = 0
	WITH [at IN nodes(path)| at.type][-3..] AS last
	MATCH (a1:Activity)-[r1]->(a2)-[r2]->(a3)-[:NEXT]->(a4)
	WHERE a1.type = last[0] 
	  AND TYPE(r2) = last[1]
	  AND TYPE(r3) = last[2]
	RETURN a4.type, COUNT(*) AS count
	ORDER BY count DESC

But that isn't much better. What if we constructed the relationship types and ran them using apoc.cypher.run:

	PROFILE MATCH path=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
    WHERE SIZE((a)-[:NEXT]->()) = 0
    WITH [at IN nodes(path)| at.type][-3..] AS last
    CALL apoc.cypher.run("MATCH (a1:Activity)-[:NEXT_" + toUpper(last[1]) +"]->(a2)-[:NEXT_" + toUpper(last[2]) +"]->(a3)-[:NEXT]->(a4) WHERE a1.type ='" + last[0] +"' RETURN a4.type AS type, COUNT(*) AS count", null) YIELD value    
    RETURN  value.type, value.count
    ORDER BY value.count DESC

That looks pretty ugly and it makes things a bit better, not really doesn't make much of a difference. We need a new approach. What if instead, we created a sort of static path index of what the next actions where for this action?

	MATCH path=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*3..]->(a)
	WITH nodes(path)[-3] AS activity, 
	REDUCE(types = nodes(path)[-3].type, n IN nodes(path)[-2..]| types + "-" +  n.type) AS key
	SET activity.next_activities = key

Let's do it for all the email addresses, and all the actions. We need emails have have performed more than 2 actions, and we will run this query until we no longer update any properties:


	MATCH (e:Email)
	WHERE SIZE((e)-[:PERFORMED]->()) > 2
	WITH e 
	MATCH (e)-[:NEXT]->(a)
	WHERE NOT EXISTS(a.next_activities)
	WITH e
	LIMIT 1000
	MATCH path=(e)-[:NEXT*3..]->(a)
	WITH nodes(path)[-3] AS activity, 
	REDUCE(types = nodes(path)[-3].type, n IN nodes(path)[-2..]| types + "-" +  n.type) AS key
	SET activity.next_activities = key

Index the next_activities property of the Activity nodes:

	CREATE INDEX ON :Activity(next_activities);

Predict what the next action for this user may be based on what others users have done in the past:

	MATCH path=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
	WHERE SIZE((a)-[:NEXT]->()) = 0
	WITH REDUCE(types = nodes(path)[-3].type, n IN nodes(path)[-2..]| types + "-" +  n.type) AS key
    MATCH (a1:Activity)-[:NEXT]->(a2)-[:NEXT]->(a3)-[:NEXT]->(a4)
    WHERE a1.next_activities = key
    RETURN a4.type, COUNT(*) AS count
	ORDER BY count DESC

What should the next action for this user be if we want to get to an opportunity quickly:

	MATCH path=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
	WHERE SIZE((a)-[:NEXT]->()) = 0
	WITH REDUCE(types = nodes(path)[-3].type, n IN nodes(path)[-2..]| types + "-" +  n.type) AS key
    MATCH path1=(a1:Activity)-[:NEXT]->(a2)-[:NEXT]->(a3)-[:NEXT]->(a4)-[:NEXT]->(a5)
    WHERE a1.next_activities = key
	  AND a5.type = "opportunity"
	RETURN a4.type, COUNT(DISTINCT path1) AS count
	ORDER BY count DESC

What should the next action for this user be if we want to get to an opportunity eventually:

	MATCH path=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
	WHERE SIZE((a)-[:NEXT]->()) = 0
	WITH REDUCE(types = nodes(path)[-3].type, n IN nodes(path)[-2..]| types + "-" +  n.type) AS key
	MATCH path1=(a1:Activity)-[:NEXT]->(a2)-[:NEXT]->(a3)-[:NEXT]->(a4), path2=(a4)-[:NEXT*]->(a5)
    WHERE a1.next_activities = key
      AND ANY (x IN nodes(path2) WHERE x.type = "opportunity")
	RETURN a4.type, COUNT(DISTINCT path1) AS count
	ORDER BY count DESC

The next_activities string property can be pretty long:

	"next_activities": "meetup-download-discovery"

In a production environment, maybe we can shorten it a bit. First remove it:

	MATCH (n:Activity) REMOVE n.next_activities		


Then use just the first two characters of the activity type for the key:

	MATCH (e:Email)
	WHERE SIZE((e)-[:PERFORMED]->()) > 2
	WITH e 
	MATCH (e)-[:NEXT]->(a)
	WHERE NOT EXISTS(a.next_activities)
	WITH e
	LIMIT 1000
	MATCH path=(e)-[:NEXT*3..]->(a)
	WITH nodes(path)[-3] AS activity, 
	REDUCE(types = LEFT(nodes(path)[-3].type,2), n IN nodes(path)[-2..]| types + "-" +  LEFT(n.type,2)) AS key
	SET activity.next_activities = key

Use the shorter version of they key:

	MATCH path=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
	WHERE SIZE((a)-[:NEXT]->()) = 0
	WITH REDUCE(types = LEFT(nodes(path)[-3].type,2), n IN nodes(path)[-2..]| types + "-" +  LEFT(n.type,2)) AS key
	MATCH path1=(a1:Activity)-[:NEXT]->(a2)-[:NEXT]->(a3)-[:NEXT]->(a4), path2=(a4)-[:NEXT*]->(a5)
    WHERE a1.next_activities = key
      AND ANY (x IN nodes(path2) WHERE x.type = "opportunity")
	RETURN a4.type, COUNT(DISTINCT path1) AS count
	ORDER BY count DESC


If we wanted to preduct their next activity with just their last two activies, we could use STARTS WITH against the next_activities property:

	MATCH path=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
	WHERE SIZE((a)-[:NEXT]->()) = 0
	WITH nodes(path)[-2].type + "-" +  nodes(path)[-1].type AS key
	MATCH path1=(a1:Activity)-[:NEXT]->(a2)-[:NEXT]->(a3), path2=(a3)-[:NEXT*]->(a4)
    WHERE a1.next_activities STARTS WITH key
      AND ANY (x IN nodes(path2) WHERE x.type = "opportunity")
	RETURN a3.type, COUNT(DISTINCT path1) AS count
	ORDER BY count DESC

What about just winning activities:

	MATCH (e:Email)
	WHERE SIZE((e)-[:PERFORMED]->()) > 2
	WITH e 
	MATCH (e)-[:NEXT]->(a)
	WHERE NOT EXISTS(a.next_winning_activities)
	WITH e
    MATCH (e)-[:PERFORMED]->(a)
    WHERE a.type = "opportunity"
	WITH DISTINCT e
	LIMIT 1000
	MATCH path=(e)-[:NEXT*3..]->(a), path2=(a)-[:NEXT*]->()
	WHERE ANY (x IN tail(nodes(path2)) WHERE x.type = "opportunity")
	WITH nodes(path)[-3] AS activity, 
	REDUCE(types = nodes(path)[-3].type, n IN nodes(path)[-2..]| types + "-" +  n.type) AS key
	SET activity.next_winning_activities = key

	CREATE INDEX ON :Activity(next_winning_activities);


	MATCH path=(e:Email {address: "drezet2@icloud.com"})-[:NEXT*]->(a)
	WHERE SIZE((a)-[:NEXT]->()) = 0
	WITH REDUCE(types = nodes(path)[-3].type, n IN nodes(path)[-2..]| types + "-" +  n.type) AS key
    MATCH path1=(a1:Activity)-[:NEXT]->(a2)-[:NEXT]->(a3)-[:NEXT]->(a4)
    WHERE a1.next_winning_activities = key
	RETURN a4.type, COUNT(DISTINCT path1) AS count
	ORDER BY count DESC

Ideas:

	// Jaccard similarity 
    CREATE (e)-[:SIMILAR_ACTIVITIES {score:xx.yy}]->(e2)
	// action path graph embedding to predict next action? (node2vec?)
