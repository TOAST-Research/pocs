# Description

The OpenWebUI server exposes an unprotected endpoint at /api/tasks/stop/{task_id} that cancels an LLM task. The endpoint accepts a task ID from the client and cancels the task without checking whether the requester actually owns the task or has the required permissions to stop it.

This allows any authenticated user (including a normal account) to stop tasks created by other users (including Admin accounts) by passing the target task's ID to the endpoint.

# Proof of Concept

## victim setup

Create two accounts on the application: one Admin and one Normal (attacker).

The Admin starts an LLM conversation or request in the web UI. This creates a server-side task that drives the model response.

Step 1 — Two user accounts (Admin and Normal):

<img width="1200" height="350" alt="Two accounts: Admin and Normal" src="https://github.com/TOAST-Research/pocs/blob/main/openwebui/arbitirary_task_stop/pics/step1_open-webui_two_accoutns.png?raw=true" />

## attack steps

1. As a Normal user, enumerate tasks by calling `/api/tasks` to obtain `task_id` values for tasks currently running on the server. The endpoint returns task objects including IDs.

	 Example (as the Normal user):

	 ```bash
	 curl -s -H "Authorization: Bearer $NORMAL_TOKEN" \
		 http://localhost:3000/api/tasks | jq .
	 ```

	 In the returned JSON, identify a `task_id` that belongs to the Admin account — this is possible because the server exposes task information without filtering ownership.

	 Step 2 — Normal user enumerates tasks and sees the Admin's task id:

	 <img width="1200" height="450" alt="Observe the task id" src="https://github.com/TOAST-Research/pocs/blob/main/openwebui/arbitirary_task_stop/pics/step2_observe_the_task_id.png?raw=true" />

2. With the `task_id` found above, the Normal user sends a POST to `POST /api/tasks/stop/{task_id}` to cancel the Admin's task.

	 Example (Normal user kills Admin's task):

	 ```bash
	 curl -X POST -H "Authorization: Bearer $NORMAL_TOKEN" \
		 http://localhost:3000/api/tasks/stop/<task_id>
	 ```

	 The server responds 200 OK and cancels the task — the Admin's LLM answer stops immediately.

	 Step 3 — Normal user kills the Admin's task (remote DoS):

	 <img width="1200" height="350" alt="Kill admin prompt / task stops" src="https://github.com/TOAST-Research/pocs/blob/main/openwebui/arbitirary_task_stop/pics/step3_success_stop_even_prompt_failing.png?raw=true" />

# Impact

By abusing this endpoint, any authenticated user can stop tasks started by other users. This leads to several impacts:

- Denial-of-Service (DoS) on specific users' requests (Admin or other target users).
- System-level DoS if many tasks are stopped in large numbers, causing disruptions and loss of service.
- Potential for abuse to interrupt critical or time-sensitive processing.

# Occurrences

The vulnerable endpoint is `POST /api/tasks/stop/{task_id}` in OpenWebUI's server API (task cancel route). The endpoint cancels the task but does not verify the authenticated user's ownership over the `task_id`.

# Remediation

Recommended fixes:

- Verify that the task belongs to the authenticated user, or that the user has explicit privilege (e.g., admin) to cancel that task. If the check fails, return HTTP 403 Forbidden.
- Restrict task listing (`/api/tasks`) to return tasks only owned by the requesting user unless the requester is an admin or has a broader view permission.
- Add unit tests for the stop endpoint to cover ownership and permission checks.
