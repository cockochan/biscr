
✅ Final Power BI Implementation Plan
1. Create a Summary Table with the Last Message
Here's a DAX summary table with just the last message per filename:
DAX
CopyEdit
LastLogPerFilename =
ADDCOLUMNS (
    SUMMARIZE ( Log, Log[filename] ),
    "LastTimestamp", 
        CALCULATE ( MAX ( Log[timestamp] ) ),
    "LastMessage",
        CALCULATE (
            FIRSTNONBLANK ( Log[content.detail], 1 ),
            FILTER (
                Log,
                Log[timestamp] = CALCULATE ( MAX ( Log[timestamp] ) )
                    && Log[filename] = EARLIER ( Log[filename] )
            )
        )
)
________________________________________
2. Flag "Not Completed" Requests
Add a calculated column:
DAX
CopyEdit
IsNotCompleted =
VAR Msg = [LastMessage]
RETURN
    IF (
        NOT (
            CONTAINSSTRING ( LOWER(Msg), "message sent" )
            || CONTAINSSTRING ( LOWER(Msg), "failed" )
        ),
        TRUE(),
        FALSE()
    )
This handles cases like:
•	"processing JSON" ✅ → not completed
•	"file received" ✅ → not completed
•	"message sent to GSC" ❌ → completed
•	"processing failed" ❌ → failed, also completed in your logic
________________________________________
3. Visualizations to Build
Now you can create visuals like:
•	Card: Total number of incomplete requests:
DAX
CopyEdit
IncompleteRequestCount = 
COUNTROWS (
    FILTER ( LastLogPerFilename, [IsNotCompleted] = TRUE() )
)
•	Table: filename, LastMessage, IsNotCompleted
•	Bar chart: Number of "not completed" requests by day (based on LastTimestamp)
________________________________________
🔧 Optional: Handle More Message Variants
If message types vary more (e.g., "message dispatched", "failed to deliver"), you could expand the logic:
DAX
CopyEdit
IsNotCompleted =
VAR Msg = LOWER ( [LastMessage] )
RETURN
    IF (
        NOT (
            CONTAINSSTRING ( Msg, "message sent" )
            || CONTAINSSTRING ( Msg, "dispatched" )
            || CONTAINSSTRING ( Msg, "failed" )
        ),
        TRUE(),
        FALSE()
    )
________________________________________
Let me know if you'd like to bundle this into a reusable Power BI measure, or turn it into a dataflow step.
You said:
i would like measure
ChatGPT said:
Great — let's create a Power BI DAX measure that counts how many requests (filenames) are not completed, based on the last log entry per filename.
________________________________________
✅ Final DAX Measure: IncompleteRequestCount
Assuming your table is called Log and it contains:
•	filename
•	timestamp
•	content.detail
Here’s the full measure:
DAX
CopyEdit
IncompleteRequestCount =
VAR LatestLogs =
    ADDCOLUMNS (
        SUMMARIZE ( Log, Log[filename] ),
        "LastTimestamp",
            CALCULATE ( MAX ( Log[timestamp] ) )
    )

VAR LogsWithLastMessages =
    ADDCOLUMNS (
        LatestLogs,
        "LastMessage",
            CALCULATE (
                MAX ( Log[content.detail] ),
                FILTER (
                    Log,
                    Log[filename] = EARLIER ( Log[filename] ) &&
                    Log[timestamp] = EARLIER ( [LastTimestamp] )
                )
            )
    )

VAR Filtered =
    FILTER (
        LogsWithLastMessages,
        NOT (
            CONTAINSSTRING ( LOWER ( [LastMessage] ), "message sent" ) ||
            CONTAINSSTRING ( LOWER ( [LastMessage] ), "failed" )
        )
    )

RETURN
    COUNTROWS ( Filtered )
________________________________________
🔍 What This Does:
1.	Gets the latest timestamp per filename
2.	Grabs the corresponding last message
3.	Filters out those that contain "message sent" or "failed"
4.	Returns the count of remaining ones — these are your "not completed" requests
________________________________________
💡 Usage
•	Use this measure in a card visual to display total incomplete requests.
•	You can also reuse the logic pattern to build other measures like:
o	A table of all incomplete requests
o	Percent completed vs not completed
