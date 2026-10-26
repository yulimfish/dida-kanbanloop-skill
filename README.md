# opencode-dida-kanbanloop

OpenCode workflow for keeping a Dida365 (滴答清单) project's Kanban board in sync with task state.

## Overview

This project explores a loop where task status and Kanban columns stay aligned:

- Active top-level tasks move to the `待办` column.
- Completed top-level tasks move to the `已完成` column.
- Subtask parent relationships are preserved; only top-level tasks decide placement.

## Notes

- Dida365 API rate limits are not publicly documented; requests should be conservative and retry after waiting 30–60 seconds when a rate-limit error occurs.
- Broad task queries may be capped to 200 results per response, so date ranges should be narrowed when enumerating history.
