JML Automation: Joiner, Mover, Leaver on ServiceNow

A ServiceNow scoped application that automates the employee access lifecycle. This repository documents the build, the design decisions and the test evidence. It is a personal project, built by Kunmi Adejumo alongside the RiseUp with ServiceNow programme as part of a move into Identity and Access Management (IAM).

Status: Joiner flow, task completion flow and Leaver flow are built and tested. The Mover flow and reporting are next (see the roadmap).

The problem

When access is managed by hand, three things go wrong:

Joiners wait days for accounts and equipment because requests are passed around by email.
Movers collect permissions from old roles that nobody removes, known as privilege creep.
Leavers keep active accounts after they have left, which is a common audit finding and a real security risk.
The solution

One request starts a repeatable, approved and auditable process. The role a person is hired into decides their standard access through a Role Template. Anything beyond the template needs manager approval.

Flow overview

What is built so far
1. Data model
Table	Purpose
Role Template (custom)	Defines standard access per job role: department, job title, default group, default catalog items
JML Event (custom)	One audit record per joiner, mover or leaver: number, type, state, employee, manager, effective date, linked request
Request, Requested Item, Catalog Task, User, Group, Group Member	Out-of-the-box tables reused for approvals and fulfilment

Reusing platform tables kept the build realistic and avoided rebuilding approvals and task handling.

Role Template fields JML Event record

2. New Starter Request (catalog item)

A Service Catalog item collects first name, last name, start date, job title (a reference to Role Template), manager and any additional access needed. A catalog client script blocks start dates in the past.

Catalog item variables

3. JML Joiner Flow

Triggered by the catalog item. In order, it:

Reads the request variables.
Looks up the Role Template matching the job title.
Creates the user, with a lowercase User ID and email built from the name.
Creates a JML Event in the In Progress state.
Looks up the template's default group and adds the user to it. Roles are inherited through the group.
Looks up the template's default catalog items and raises one fulfilment task per item for the fulfiller group.
Sends a notification to the manager.
If additional access was requested, asks the manager for approval. A rejection sets the event to Partially Complete.

Group and task steps Approval branch

4. JML Task Completion flow

A second flow triggers when a catalog task is closed complete. It checks whether any tasks for the same request are still open. When none remain, it sets the JML Event to Complete.

Task completion flow

5. Leaver Request (catalog item)

A Service Catalog item collects the employee, the leaving date and an Immediate departure checkbox. The event needs an effective date, so an immediate departure simply uses today's date.

Leaver catalog item variables

6. JML Leaver Flow

Triggered by the Leaver Request. In order, it:

Reads the request variables.
Looks up the user record for the leaving employee.
Creates a JML Event of type Leaver in the In Progress state, linked to the request.
Checks whether the departure is immediate. A scheduled departure waits until the leaving date, and an immediate departure skips the wait.
Looks up every group membership the user holds, so the number of memberships is known.
Deactivates the account instead of deleting it, so the record stays available for audit and sign-in is blocked straight away.
Raises a task for the fulfiller group to remove the group memberships.
Raises an asset recovery task for the fulfiller group.
Sends a notification to the manager.

The existing task completion flow closes the Leaver event when both tasks are done, so no extra closing logic was needed.

Group removal is a fulfiller task and not an automated delete. See the design decisions for why.

Leaver flow overview Leaver event created Wait branch Revocation steps

Design decisions
Access through groups, not direct role grants. Users get roles by joining a group, so access follows group membership. This is standard IAM practice and is easier to review and revoke than person-by-person grants.
Standard access is not held back by approvals. Template access is granted immediately and only the extra request waits for the manager. A rejection therefore leaves the event Partially Complete, not Cancelled, because the joiner itself is not undone.
Fulfilment tasks do not pause the main flow. The flow does not wait on each task. A separate flow closes the event when the last task finishes, so the approval and notification steps are never blocked by equipment tasks.
The flow runs as the system user. Creating users and memberships does not depend on who submitted the request, so access to the app is controlled by roles and catalog availability instead.
Every action leaves a record. The JML Event, approval records and flow execution history together give an audit trail of who requested what and what the system did.
Leavers are deactivated, not deleted. Keeping the user record preserves the history that approvals, tasks and events point to, so the account can still be audited after the person has gone.
The audit event is logged before any access is removed. If a later step fails, the request and its owner are already on record.
Scheduled and immediate departures share one flow. The only difference is the wait step, so both paths run the same revocation steps and behave consistently.
Group removal is a task, not an automated delete. The platform refused to let the app delete Group Member records ("Scope does not have delete access to table sys_user_grmember"), even with a cross-scope privilege and table level delete enabled. The account is deactivated automatically, which blocks sign-in, and a task is raised for the fulfiller group to remove the memberships. This also keeps a person in the loop on access removal. The planned orphaned account check will flag any inactive user who still holds a membership, so a missed task cannot go unnoticed.
Tasks never pause the Leaver flow. The task actions do not wait for completion, so the manager notification and the second task always run. The task completion flow closes the event once every task is done.
Testing
#	Scenario	Expected result	Evidence	Result
1	Standard joiner, no extra access	User, group membership, event (In Progress), tasks and manager notification created; approval branch not run	Execution, User groups, Tasks	Passed
2	Close all fulfilment tasks	Event stays In Progress until the last task closes, then moves to Complete	One task closed, Both closed	Passed
3	Extra access approved	Flow pauses at approval, resumes on approval, event stays In Progress until tasks close	Waiting, Pending approval, Approved	Passed
4	Extra access rejected	Rejection branch runs and the event becomes Partially Complete	Execution, Event	Passed
Leaver flow scenarios
#	Scenario	Expected result	Evidence	Result
L1	Immediate leaver	Wait skipped, user inactive, group removal task and recovery task created, event In Progress	Before, Execution, After, Tasks	Passed
L2	Close both tasks	Event stays In Progress until the last task closes, then moves to Complete	Recovery task, Event complete	Passed
L3	Scheduled leaver (future date)	Run waits, user keeps access until the leaving date	Waiting	Passed
L4	Leaving date is today	Wait passes at 08:00 on the day, same outcome as L1	Waiting, Execution	Passed

A date-only leaving date resolves to 08:00 on this instance. In L4 the run was ordered at 05:02, waited, and the timer completed at 08:00:00. The remaining steps then ran at 08:00:47 with the same outcome as L1.

Scenario 1 evidence: access granted through the group

The new user was added to the Finance Team group by the flow, and the group's roles arrived through inheritance.

User groups User roles, inherited from the group

Scenario 2 evidence: the event completes only when the last task closes

With one of two fulfilment tasks closed, the event correctly stays In Progress.

One task closed, event still In Progress

Once the second task is closed, the task completion flow sets the event to Complete.

Both tasks closed, event Complete

Scenario 3 evidence: extra access approved

With extra access requested, the flow granted standard access first, then paused at the manager approval.

Flow paused at Ask For Approval

The pending approval was assigned to the manager.

Pending approval record

After the manager approved, the same run resumed and completed. The event stayed In Progress until the fulfilment tasks were closed.

Run completed after approval

Scenario 4 evidence: extra access rejected

When the manager rejected the extra access request, the rejection branch ran and the Update step completed.

Rejection branch executed

The event was set to Partially Complete, because the standard access had already been granted and only the extra request was refused.

Event showing Partially Complete

Problems solved along the way
Problem	Cause	Fix
Template lookup found nothing	The Reference variable passed the display text, but the condition matched on Sys ID	Matched on the Job Title field instead
Catalog variables arrived empty	Test runs used a request item that held no variable values	Tested with real catalog orders
List fields rejected in a condition	A Glide List field cannot be dropped into an "is one of" condition	Used a scripted encoded query (sys_idIN plus the list value)
Flow never started from a real order	The catalog item was not linked to the flow	Set the flow on the item's Process Engine tab
Role table not available in the flow	Platform table not exposed to the scoped app	Granted roles through group membership instead
Approval step ran on every outcome	The update step was outside the rejection branch	Moved it under the branch so it only runs on rejection
Event completed too early	The flow paused on each task	Removed the wait and closed the event from a separate flow
Leaver lookup returned no user	The employee reference pill passes a sys_id, but the condition matched on the Name field	Matched Name against the expanded Name child of the pill, or Sys ID against the top-level pill. Changing the condition field clears the value, so it has to be re-dropped
Wait ran for immediate leavers	The wait step sat outside the If, so it ran on every path	Moved it inside the If's "then" branch, using a relative duration of zero after the leaving date
Group memberships could not be deleted	The platform blocked delete on Group Member from a scoped app, despite a cross-scope privilege and table level Can delete	Deactivated the account automatically and raised a fulfiller task to remove the memberships. See the design decisions
Flow stuck at Waiting after the group removal task	The Create Catalog Task action was set to wait for the task to complete	Turned off the wait so the flow carries on and the completion flow closes the event
Roadmap
 Mover flow: grant the new role's access, then create a time-boxed task for the manager to confirm removal of the old access
 Leaver flow: deactivate the account, raise group removal and asset recovery tasks, with an immediate path and a scheduled path- [ ] Orphaned account check: daily scheduled flow that flags inactive users who still hold group memberships
 Access controls: restrict who can create or change templates and events, and enforce separation of duties on approvals
 Dashboard: open reviews, task completion times and overdue items
 Integration design: diagram showing how the flows would connect to Active Directory or Entra through a MID Server
Skills demonstrated

ServiceNow scoped application design, Service Catalog, Flow Designer, data modelling, approvals, role and group based access, joiner-mover-leaver governance, testing against defined scenarios, and troubleshooting through flow execution logs.

Notes

All users, names and email addresses in this project are test data on a ServiceNow Personal Developer Instance. No real personal data or credentials are stored in this repository.
