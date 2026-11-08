Follow the latest authoritative Admin-Manual through KStore. This repository is the independent clean-room Natally rewrite.

Preserve the complete intended functionality and the original look and sound. Reuse original mascot/media/voice assets; write application code anew. Do not copy old application implementation, historical patch plans or secrets. Do not modify the original repository or live deployment as part of this rewrite.

ARCHITECTURE.md first: DOCS/ARCHITECTURE.md describes the finished system and enumerates every entity. CHECKLIST.md translates it faithfully. Keep every requirement and constraint together; never silently narrow scope.

No placeholder functionality, mock customer experiences, dummy payments, or invented customer data. Credentials remain in canonical storage; .env contains only kstore:// pointers for secrets.

Operator-directed verification: run the actual application journeys and record them. No separate review-agent cycles or unit/smoke test work. Do not claim completion from static inspection alone.

Work only within assigned file ownership. Do not let unrelated tracks collide. An explicit stop halts agents immediately. Commit and push granular completed work, preserve rollback paths, and never interpret an active correction as operator absence.

Commit and push to `origin` after EVERY completed task and EVERY turn that changes files. Checkpoint unfinished changes explicitly; never label a checkpoint as completed functionality. Stage only the intended files, never credentials, runtime secret material or unrelated work. Report a failed push accurately and retain the local commit for retry; do not claim remote delivery until the push succeeds.
