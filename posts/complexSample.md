---
title: Comp
date: 2020-01-12
---

The new tools for the complex agent should be like now I am gonna identify the data I need, k I did nt find the data I need. 

Now, I'm going to go do a similar Search for tables. Do I have the data I need, no?

Now, I'm going to go to search for columns right. 

And so we proposing, we can do this concurrently or in parallel.

Similarity searches. 

Get Queries. 

Get Raw tables not reporting tables. 

Business rules 

get measures

AĹL into one tool. 

It goes to the next step to, like, find the joins and stuff because I'm like, can't really do anything with join information until it knows like, what tables it's actually going to use. 

So if it didn't get the data I need before the joins, what does it do like? 

Are we just Gonna go Back to the user to ask follow up questions? 

Would it just like do another similarity search, Uh, maybe rephrase the question A little bit, engaging the user wt an approval button for deeper analysis. 

\# Complex Agent — Tool Architecture Analysis & Gaps

\#\# Foundation: What Already Exists

\`complex\_agent.py\` has a working semantic layer (\`\_SEMANTIC\_LAYER\`, lines 52–169) and three semantic discovery tools that directly match what the PDF describes:

| PDF Recommendation | Existing in Codebase |  
|---|---|  
| \`semantic\_search(query)\` | \`semantic\_search\_on\_semantic\_layer(query)\` — keyword scoring against the layer |  
| \`get\_column\_metadata(tables)\` | \`retrieve\_semantic\_layer\_details(table\_name)\` — returns full column docs |  
| \`find\_relevant\_tables(query)\` | Embedded in \`semantic\_search\_on\_semantic\_layer\` — returns \`recommended\_tables\` |  
| Measures tool | \`metric\_formulas\` block inside each table entry (e.g., \`total\_revenue\`, \`labor\_cost\_ratio\`) |  
| Business rules tool | \`common\_filters\` inside each table entry (partial — filter patterns only) |  
| Join metadata | \`join\_keys\` block inside each table entry (1-hop only) |

The \`\_SEMANTIC\_LAYER\` dict \*\*is\*\* the semantic layer. It's just not yet broken out into focused tools.

\---

\#\# What's MISSING (Gaps to Fill)

\#\#\# 1\. No \`gather\_complex\_context(query)\` Composite Tool  
The PDF's core recommendation. Currently the agent calls \`semantic\_search\` → \`retrieve\_semantic\_layer\_details\` → \`get\_table\_info\` \*\*sequentially\*\*. There is no single tool that fans out \*\*concurrently\*\* to get semantic matches \+ measures \+ business rules \+ joins in one call.

\*\*What it should return:\*\*  
\- semantic matches  
\- candidate tables  
\- candidate measures  
\- business rules  
\- possible joins (if table set is obvious)  
\- confidence / similarity metadata

\#\#\# 2\. No \`get\_measure\_definition()\` Tool  
Measures are buried *\*inside\** \`retrieve\_semantic\_layer\_details()\` alongside 20+ column docs. The agent cannot ask "what is labor cost ratio?" without loading the entire table's context.

The \`metric\_formulas\` dict is the data — it just needs its own tool surface.

\*\*Examples of measures that need to be independently queryable:\*\*  
\- \`net\_sales\`  
\- \`gross\_sales\`  
\- \`labor\_cost\_ratio\`  
\- \`avg\_order\_value\`  
\- \`covers\_per\_seat\`

\#\#\# 3\. No \`get\_business\_rules()\` Tool — BIGGEST GAP  
\`common\_filters\` covers filter patterns only (e.g., \`{"channel": "filters={'fulfillment\_type': 'delivery'}"}\`). There is \*\*nothing\*\* encoding actual domain rules like:

\- "voided tickets are excluded from revenue"  
\- "fiscal week starts Monday"  
\- "active location \= has orders in last 30 days"  
\- "refunds are netted out of revenue"  
\- "delivery orders exclude the platform fee"

*\> Wrong SQL outputs are often \*\*business-logic errors, not syntax errors.\*\* This is the most impactful missing piece.*

\#\#\# 4\. No \`find\_join\_paths(tables)\` Tool  
\`join\_keys\` only has direct 1-hop joins (e.g., \`orders → locations\` on \`location\_id\`). Multi-hop is not modeled.

The \`staff\_schedule → orders\` join is documented as free text:  
\`\`\`  
"location\_id (match schedule\_date ≈ order\_date for the same day)"  
\`\`\`  
This is not queryable. There is no confidence score or "indirect join path" output.

\*\*What this tool should return:\*\*  
\- known join keys  
\- confidence / source  
\- possible join path if indirect  
\- surface missing join metadata clearly

\#\#\# 5\. Semantic Search is Keyword Scoring, Not Vector Similarity  
\`semantic\_search\_on\_semantic\_layer\` does simple word matching:  
\- \+2 per word hit in description  
\- \+3 for metric key match  
\- \+1 for column match

Rephrased follow-up questions ("what about last week?") will miss unless exact words overlap. This is the \*\*follow-up question context loss\*\* problem described in the PDF.

\#\#\# 6\. No \`if data not found → fallback\` Logic  
When similarity search finds nothing, the agent currently returns all 5 tables as a fallback (line 229). There is no structured path to:

1\. Rephrase the query slightly  
2\. Retry similarity search  
3\. Surface a clarification prompt to the user

\---

\#\# Recommended Tool Set (from PDF)

\#\#\# Minimum tool set for the complex agent:

\`\`\`  
gather\_complex\_context(query, conversation\_context=None)  
  → runs concurrently inside:  
      \- semantic similarity lookup  
      \- measures lookup  
      \- business rules lookup  
      \- maybe table lookup

find\_join\_paths(tables)  
expand\_measure\_definition(name)  
get\_rule\_details(rule\_name)  
\`\`\`

\#\#\# Recommended agent flow:

1\. Rewrite current turn into a \*\*standalone question\*\* if needed  
2\. Run \*\*similarity search\*\* against the semantic layer  
3\. Pull relevant \*\*measure definitions\*\*  
4\. Pull relevant \*\*business rules\*\*  
5\. Find tables / columns / joins  
6\. Construct SQL plan  
7\. Validate against gathered semantic context

\---

\#\# Where to Build From

| What to build | Build from |  
|---|---|  
| \`gather\_complex\_context()\` | Wrap existing \`semantic\_search\_on\_semantic\_layer\` \+ new concurrent lookups |  
| \`get\_measure\_definition()\` | Extract \`metric\_formulas\` from \`\_SEMANTIC\_LAYER\` into own tool |  
| \`get\_business\_rules()\` | Add \`business\_rules\` section to \`\_SEMANTIC\_LAYER\`, expose as tool |  
| \`find\_join\_paths()\` | Extract \+ expand \`join\_keys\` from \`\_SEMANTIC\_LAYER\`, add multi-hop \+ confidence |  
| Semantic search upgrade | Replace keyword scoring with vector embeddings |

\*\*Core file:\*\* \`complex\_agent.py:52–289\`  
\*\*Data model to extend:\*\* \`\_SEMANTIC\_LAYER\` dict — add \`business\_rules\` section per table  
\*\*Pattern:\*\* Not a rewrite — a refactor \+ gap-fill

\---

\#\# Key Principle (from PDF)

*\> Use the \*\*same semantic layer\*\* as the source, but expose it through focused tools:*  
*\> \- one tool for similarity retrieval*  
*\> \- one for measures*  
*\> \- one for business rules*  
*\> \- one for joins*  
*\>*  
*\> That way the complex agent can retrieve \*\*only what it needs at each step.\*\**

Thread pool / concurrency should happen \*\*inside tool execution\*\*, not as hidden prefetch before the agent runs.

