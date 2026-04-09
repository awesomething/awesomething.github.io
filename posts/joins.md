---
title: Collection of Joins
date: 2021-12-02
---

JOINS CONVO  
The join tool for the complex agent, so essentially the complex agent is, you know. Yeah, that's really what sets it apart from the KPI agent. Yeah, somebody needs to fix that today. So what do you want me to fix? I can take it over, but I just want to get specifics. So the tool, there's something wrong with the retrieval part in the tool. So like you should only need to fix the join logic in the semantic layer tool file. Like the get join tool that it's calling is just, all it's returning, when I looked at it was, like it wasn't returning data that was relevant. It returned something, but it's not relevant to the request. 

Like it should return all the joins for that the table that's requested. I would, like, let's err on the side of like, try to make it work for how the semantic layer is structured today. Because it, it should just be like a simple. Like the join logic is separated by like left join, like left table, right table in our semantic layer. And so the logic should be like the AI passes the table name, and if it's the left table or the right table for that table name, we want to return the info. But there's just something wrong. 

I don't know what it is, but I've noticed. So just test it, like force the agent to call that tool. I think it's get join or something. Okay. And this, like we need this one today. Okay. Should be working. Yeah, that's 24 tables. So this is not working for some reason. ANY RECOMMENDATIONS? also SUMMARIZE

Here’s the gist of what needs to happen.

**Summary**  
The issue is in the **join retrieval logic** for the **complex agent’s semantic layer tool**, not the KPI agent. The tool being called, likely **`get_join`**, is returning results, but the results are **not relevant to the requested table**. The expected behavior is simple: when the AI passes a table name, the tool should return **all joins where that table appears either as the left table or the right table** in the semantic layer. Since the semantic layer currently stores joins as **left\_table / right\_table pairs**, the fix should follow that structure rather than redesigning anything. There are about **24 tables**, and this needs to be fixed **today**.

**Recommendations**

1. **Inspect the semantic layer tool file first**  
   Focus on the join logic in the semantic layer tool, especially the function that backs `get_join`.  
2. **Check the filter condition**  
   The likely bug is that it is:  
   * only checking one side of the join, or  
   * doing an incorrect match against the requested table name, or  
   * returning a broader/default set instead of filtering correctly.

The matching logic should effectively be:  
join.left\_table \== requested\_table or join.right\_table \== requested\_table

3.   
4. **Normalize table names before matching**  
   To avoid false misses:  
   * trim whitespace  
   * lowercase both values  
   * handle aliases only if the semantic layer already supports them  
5. **Return all matching joins, not just one**  
   Make sure the tool collects every relevant join for the table, since one table may join to multiple others.  
6. **Test by forcing the agent to call the tool**  
   Don’t just unit test the helper. Run the complex agent in a way that guarantees it calls `get_join`, then verify the output for a few known tables.  
7. **Test both left-side and right-side cases**  
   Pick examples where the requested table appears:  
   * as `left_table`  
   * as `right_table`  
   * in multiple joins  
   * in no joins  
8. **Avoid over-engineering**  
   Since this is needed today, keep the fix aligned with the current semantic layer structure instead of refactoring the model.

**Likely root cause**  
Most likely, the retrieval logic is only checking one join direction or is using the wrong field when filtering, which causes irrelevant or incomplete results.

**Suggested implementation behavior**  
Given a table name, the tool should:

* scan all join definitions  
* return every join where the table is on either side  
* include enough join metadata for the agent to use it meaningfully

**One-sentence summary**  
Fix the `get_join` logic in the semantic layer tool so that, for any requested table, it returns **all joins where that table is either the left or right table**, then validate it by forcing the complex agent to call the tool against several of the 24 tables.

I can also turn this into a cleaner Slack update or engineer handoff note.

