# Recipe Manager: interview prep

## 30-second pitch

Recipe Manager is a Flask and MongoDB Atlas web app where users register, log in and manage a private recipe
collection with full create, read, update and delete. I containerized it with Docker and Gunicorn, pushed the image to
Amazon ECR and deployed it on AWS ECS Fargate. The database URI and session key live in AWS Secrets Manager and are
injected at container start, so the image never contains a secret, and logs go to CloudWatch.

## 2-minute walkthrough

1. **App.** Flask with Jinja2 templates and Bootstrap 5. Flask-Login handles sessions and protects every recipe route
   with `@login_required`. Passwords are hashed with Werkzeug's `generate_password_hash`.
2. **Data.** Two MongoDB collections, `users` and `recipes`. Each recipe stores its owner's ID in `user_id`, and every
   query filters on the current user, so one user cannot read or change another user's recipes. A recipe that belongs
   to someone else returns 404.
3. **Config.** `SECRET_KEY` and `MONGO_URI` are read from the environment. Locally they come from a git-ignored
   `.env`; `.env.example` documents them.
4. **Container.** `python:3.12-slim`, pinned `requirements.txt`, Gunicorn with 2 workers, 2 threads and a 60-second
   timeout on port 5000. `.dockerignore` keeps `.env` and the virtualenv out of the image.
5. **Publish.** Create an ECR repository, log Docker in, tag and push `recipe-manager:latest`, then confirm it with
   `aws ecr describe-images`.
6. **Deploy.** Store both secrets in Secrets Manager, register a Fargate task definition (256 CPU units, 512 MiB,
   `awsvpc`, secrets by ARN, `awslogs` driver), create a cluster, a security group for port 5000 and a service with a
   desired count of 1 and a public IP.
7. **Lessons.** Quoted values in `.env` break Docker's `--env-file`; one sync Gunicorn worker was fragile, so I moved
   to 2 workers with threads; an open port attracts scanners; Chrome's HTTPS-first mode makes plain HTTP look broken;
   and a TLS alert from Atlas means the network path already works.
8. **Next.** An Application Load Balancer with HTTPS, CI/CD with commit-SHA image tags, CSRF protection and tests.

## Questions and answers

### Overview

**1. What does the app do?**
Users register and log in, then create, view, edit and delete their own recipes. Each recipe has a title, ingredients,
steps, a category from 7 options and a cuisine from 10 options.

**2. What is the deployment pipeline in one line?**
Docker image, then Amazon ECR, then Secrets Manager for the two secrets, then an ECS task definition, then an ECS
cluster and Fargate service, with MongoDB Atlas as the external database.

**3. What changed between the local app and the deployed one?**
Secrets moved from code to environment variables, the UI was redesigned, the app was containerized with Gunicorn, and
the AWS pipeline was added. Persistent data was already outside the app in Atlas.

### Design decisions

**4. How do you keep users' recipes separate?**
Every recipe document stores `user_id`, and every read, update and delete filters on
`{"user_id": current_user.id}`. Update and delete first load the recipe with that filter and return 404 if it is not
found, so another user's ID behaves exactly like a missing one.

**5. Why MongoDB?**
A recipe is a self-contained document with no joins needed beyond the owner, and Atlas is managed. A relational
database would also work; the README notes that the user link is enforced in application code, not by the database.

**6. How are passwords stored?**
As salted hashes from `werkzeug.security.generate_password_hash`; login uses `check_password_hash`. The plain password
is never stored.

**7. How do sessions work across multiple containers?**
Flask-Login keeps the user ID in Flask's session cookie, which is signed with `SECRET_KEY` and stored in the browser.
Every task gets the same `SECRET_KEY` from Secrets Manager, so any task can validate any cookie without a shared session
store.

**8. Why does delete have a confirmation page?**
GET on `/delete_todo/<id>` renders a confirmation page and only a POST deletes, so following a link cannot delete a
recipe by itself.

### Docker and Gunicorn

**9. Walk me through the Dockerfile.**
Start from `python:3.12-slim`, set `/app` as the working directory, copy and install `requirements.txt` first so the
dependency layer is cached, copy the code, expose 5000 and start Gunicorn with
`--workers 2 --threads 2 --timeout 60 app:app`.

**10. Why Gunicorn and not `python app.py`?**
`app.run` is Flask's development server and runs with `debug=True` in this file. Gunicorn is a production WSGI server
with multiple workers and a request timeout. The `__main__` block only runs locally.

**11. Why 2 workers and 2 threads?**
With the default single sync worker, one slow or half-open connection, such as a port scanner, could block the app
until the worker timed out. Two workers with two threads each can serve up to four requests at once on a small
Fargate task.

**12. What is the `.env` quoting problem?**
python-dotenv strips quotes around values, Docker's `--env-file` does not. A quoted `MONGO_URI` passed a literal quote
character into the connection string and PyMongo rejected it.

### AWS deep dive

**13. Why ECR and not Docker Hub?**
ECR is private and ECS pulls from it with IAM permissions, so no registry password is needed and the image is not
public.

**14. How do secrets get into the container?**
The task definition's `secrets` block maps `MONGO_URI` and `SECRET_KEY` to Secrets Manager ARNs. When the task starts,
ECS uses the execution role to read them and sets them as environment variables. The values never appear in the image
or in the task definition.

**15. Why Secrets Manager and not plain environment variables in the task definition?**
Plain values in a task definition are visible to anyone who can describe it. Secrets Manager encrypts the values and
logs access, and the task definition holds only ARNs.

**16. What is in the task definition?**
Family `recipe-manager-task`, `FARGATE` compatibility, `awsvpc` networking, 256 CPU units and 512 MiB, the execution
role, one container with the ECR image, port 5000, the two secrets and `awslogs` logging to `/ecs/recipe-manager`.

**17. What is the difference between the cluster, the service and the task?**
The cluster is a logical namespace. The task is one running copy of the task definition. The service keeps the desired
number of tasks running and replaces any that stop.

**18. What does `awsvpc` mode give you?**
Each task gets its own network interface and private IP, and with `assignPublicIp=ENABLED` a public IP. Security
groups attach to the task directly.

**19. How did you find the app's address?**
`aws ecs list-tasks`, then `describe-tasks` to get the network interface ID, then `aws ec2 describe-network-interfaces`
for the public IP.

### Scaling and failure modes

**20. What happens if the task crashes?**
The service starts a replacement because the desired count is 1. All data is in Atlas, so nothing is lost, but the new
task gets a new public IP and there is a short outage.

**21. How would you scale it?**
Put an Application Load Balancer in front, run two or more tasks across availability zones and add target-tracking
autoscaling. Sessions and data are already outside the container, so tasks are interchangeable.

**22. What would you fix first for production?**
HTTPS through an ALB with an ACM certificate, a stable DNS name, and a security group that only accepts traffic from
the ALB. Then CSRF protection, a unique index on `username` and a health check route.

**23. Where are the security gaps today?**
Plain HTTP, port 5000 open to the internet, no CSRF tokens on forms, logout over GET, username uniqueness checked in
code rather than by an index, and the container runs as root.

**24. How would you deploy without typing CLI commands?**
A CI/CD pipeline that builds the image, tags it with the commit SHA instead of `latest`, pushes to ECR, registers a new
task definition revision and updates the service, so every deploy is traceable and easy to roll back.

**25. How would you know the app is healthy?**
Today only the CloudWatch logs. I would add a `/health` route, ALB health checks and CloudWatch alarms. Response time
and uptime are not measured yet: p95 latency —, uptime —.

### Debugging scenarios

**26. The container cannot connect to MongoDB and you see a `ServerSelectionTimeoutError` with a TLS alert. Where do
you look?**
A TLS alert from Atlas means the network path already worked: security group, routing and DNS are fine. The rejection
happened at the TLS or Atlas layer, so check the connection string and the Atlas side, such as network access rules,
instead of IAM or security groups.

**27. The app works in Firefox but hangs in Chrome. Why?**
Chrome's "Always use secure connections" tries to upgrade `http://` to HTTPS, and there is no TLS listener on the
port, so the connection hangs. The fix is HTTPS through an ALB and ACM.

**28. The task stops right after starting. How do you debug it?**
`aws logs tail /ecs/recipe-manager` and `describe-tasks` for the stopped reason. Common causes: the execution role
cannot read the secrets, the image tag does not exist in ECR, or a missing variable. `app.py` reads `SECRET_KEY` and
`MONGO_URI` with `os.environ[...]`, so a missing one fails at startup with a `KeyError`.

**29. The app occasionally freezes for 30 seconds. What happened?**
This happened with a single sync Gunicorn worker: a connection that opened a socket and sent nothing blocked the worker
until the default 30-second timeout. Moving to 2 workers with 2 threads and a 60-second timeout fixed it.

### Behavioral (STAR outlines)

**30. Tell me about a deployment problem you solved.**
- Situation: the container could not connect to MongoDB after deployment.
- Task: find which layer was failing.
- Action: read the error closely; it was a TLS alert from Atlas, which meant the AWS network path already worked, so I
  stopped checking IAM and security groups and looked at the connection and Atlas side.
- Result: fixed the right layer instead of changing working AWS settings; time saved —.

**31. Tell me about improving reliability.**
- Situation: the app sometimes stopped responding when a client held a connection open.
- Task: keep it responsive.
- Action: identified the single sync Gunicorn worker as the bottleneck and moved to 2 workers, 2 threads and a
  60-second timeout.
- Result: one slow connection no longer blocks the app; before and after numbers —.

**32. Tell me about handling secrets.**
- Situation: the app needed a database URI and a session key in production.
- Task: keep them out of the code, the repository and the image.
- Action: environment variables locally through a git-ignored `.env`, `.dockerignore` to keep it out of the image, and
  Secrets Manager ARNs in the task definition for AWS.
- Result: the same image runs locally and on Fargate and contains no secret.

## Resume bullet templates

- Built a Flask and MongoDB Atlas recipe manager with Flask-Login authentication, hashed passwords and per-user CRUD
  access enforced on every query.
- Containerized the app with Docker/Gunicorn, pushed to Amazon ECR and deployed on AWS ECS Fargate with Secrets Manager
  injecting the database URI and session key at container start, and CloudWatch Logs for monitoring.
- Diagnosed production issues across layers (Docker env-file quoting, Gunicorn worker blocking, Atlas TLS errors) and
  documented the deployment in a step-by-step write-up; p95 latency — and uptime — (fill in after measurement).
