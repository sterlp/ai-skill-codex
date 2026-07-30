Implement the plan.
                    
If the plan is not already saved, save it using a filename derived from the feature.
Treat the plan file as long-term memory — update it as decisions are made or steps completed.

If the plan contains several tasks - or is large splitt to to tasks:
- Create a separete task file for each individual task / rule 
- implement and work on them individually.

When switching to a different piece of work:
1. Batch in parallel: run compactSession on the current conversation + read the plan file + read any referenced files or prior plans.
2. Pass into the preserve parameter: this handover instruction, the plan file path, and the next steps.