# DevOps Consolidation Exercise

A hands-on exercise covering CI/CD and Infrastructure as Code, designed for use in GitHub Codespaces.

By the end of this exercise you will have a working pipeline that:
- Runs automated tests on every push
- Builds and smoke-tests a Docker image
- Demonstrates the `needs` dependency between jobs (build only runs if tests pass)

---

## Before You Start

### What's in this repo?

| File | What it is |
|---|---|
| `app.py` | A minimal Flask web application |
| `test_app.py` | Automated tests for the app |
| `requirements.txt` | Python dependencies |
| `Dockerfile` | Infrastructure as Code — defines the container |
| `docker-compose.yml` | Infrastructure as Code — defines the service |
| `.github/workflows/pipeline.yml` | The CI/CD pipeline |
| `.devcontainer/devcontainer.json` | Codespaces configuration (enables Docker-in-Docker) |

### DevOps Stories — read these first

This exercise is framed around **DevOps Stories**: the "As a [role], I want [capability], so that [benefit]" format. Each step below has a story attached. As you work through the exercise, think about how each story connects to the pipeline component you're building.

---

## Step 0 — Open in Codespaces

1. Push this repo to your GitHub account (or fork it)
2. Click **Code → Codespaces → Create codespace on main**
3. Wait for the container to build — this takes a couple of minutes on first launch as it installs Docker-in-Docker
4. Once the terminal is available, verify Docker is working:

```bash
docker --version
docker compose version
```

> **Troubleshooting:** If `docker` is not found, the container may not have finished building. Try: Ctrl+Shift+P → "Codespaces: Rebuild Container".

---

## Step 1 — Run the app locally

> 🗒️ **DevOps Story:** *"As a developer, I want to run the application locally inside a container, so that I know my code works in an environment identical to production."*

Start the app with Docker Compose:

```bash
docker compose up --build
```

You should see Flask start up and confirm it's listening on port 5000. Codespaces will prompt you to open the forwarded port — click **Open in Browser** and you should see:

```
Hello from my DevOps pipeline!
```

Also test the health endpoint:

```bash
curl http://localhost:5000/health
```

Expected response:
```json
{"status": "ok"}
```

Stop the app when you're done:

```bash
docker compose down
```

**Reflection question:** What would a new team member have to do to run this application if the `Dockerfile` and `docker-compose.yml` didn't exist?

---

## Step 2 — Run the tests locally

> 🗒️ **DevOps Story:** *"As a developer, I want to run the test suite locally before pushing, so that I can catch issues before the pipeline runs."*

Install the dependencies and run pytest directly:

```bash
pip install -r requirements.txt
pytest test_app.py -v
```

You should see both tests pass:

```
test_app.py::test_home PASSED
test_app.py::test_health PASSED
```

**Reflection question:** Open `test_app.py` and `app.py` side by side. Can you trace how each test maps to a route in the application?

---

## Step 3 — Trigger the CI/CD pipeline

> 🗒️ **DevOps Story:** *"As a developer, I want my code automatically tested and built on every push, so that I get fast feedback without having to remember to run anything manually."*

Make a small change to the application — for example, update the greeting in `app.py`:

```python
return "Hello from my DevOps pipeline! 🚀", 200
```

Commit and push:

```bash
git add .
git commit -m "Update greeting message"
git push
```

Then go to your repository on GitHub and click the **Actions** tab. You should see the pipeline running.

Watch the two jobs:

1. **Run Tests** — installs Python, runs pytest
2. **Build Docker Image** — builds the image and runs a smoke test against the `/health` endpoint

Notice that the **Build** job has a `needs: test` dependency — it will not start until the **Test** job completes successfully.

**Reflection question:** Why is this ordering important? What would happen if the build ran before the tests?

---

## Step 4 — Break the pipeline (deliberately)

> 🗒️ **DevOps Story:** *"As a team lead, I want failed pipelines to block broken code from proceeding, so that the main branch stays deployable at all times."*

Add a failing test to `test_app.py`:

```python
def test_broken():
    assert 1 == 2
```

Commit and push, then watch the Actions tab again. You should observe:

- The **Test** job fails
- The **Build** job never starts — it's blocked by the failed dependency
- The pipeline run is marked as failed overall

Now revert the broken test, push again, and confirm the pipeline goes green.

**Reflection question:** In the context of your summative assessment, how would you narrate this behaviour as a DevOps Story in your demonstration?

---

## Step 5 — Run the full stack manually in Codespaces

> 🗒️ **DevOps Story:** *"As a DevOps engineer, I want infrastructure provisioned as code, so that environments are consistent and reproducible without manual setup steps."*

Try building and running the container directly (without Compose) to see what the pipeline's `build` job does:

```bash
docker build -t my-app:local .
docker run -d --name my-app -p 5000:5000 my-app:local
curl http://localhost:5000/health
docker stop my-app
docker rm my-app
```

**Reflection question:** The pipeline tags the image with `${{ github.sha }}` — the commit hash. Why is this useful? How does it relate to rollback capability?

---

## Extension Tasks

If you finish early, try one or more of the following:

### A — Add a new route and test

Add a `/version` route to `app.py` that returns a JSON object with a version number. Write a corresponding test in `test_app.py`. Push and confirm the pipeline stays green.

### B — Introduce a linting step

Add a `lint` job to `.github/workflows/pipeline.yml` that runs before `test`, using `flake8`:

```yaml
lint:
  name: Lint
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-python@v5
      with:
        python-version: "3.12"
    - run: pip install flake8
    - run: flake8 app.py test_app.py
```

Update the `test` job to include `needs: lint`. Observe the new job ordering in the Actions tab.

### C — Write your own DevOps Stories

For each component in this repo, write a DevOps Story in the format:

> *"As a [role], I want [capability], so that [benefit]."*

Think about: the `Dockerfile`, `docker-compose.yml`, `pipeline.yml`, the `needs: test` dependency, and the smoke test step. Aim for 5–8 stories — the same number required in your summative demonstration.

---

## Connecting to Your Summative Assessment

| Pipeline component | Assessment criterion |
|---|---|
| Automated tests (`pytest`) | Practical skills — demonstrating CI |
| `Dockerfile` + `docker-compose.yml` | Design & architecture — IaC justification |
| `needs: test` job dependency | Knowledge & understanding — explaining why |
| Deliberate failure → green recovery | Reflection on practice — real-world relevance |

When recording your demonstration, use the DevOps Stories format to narrate each component. Don't just show what happens — explain the problem it solves and who benefits.

---

## File Structure Reference

```
.
├── .devcontainer/
│   └── devcontainer.json       # Enables Docker-in-Docker in Codespaces
├── .github/
│   └── workflows/
│       └── pipeline.yml        # CI/CD pipeline definition
├── app.py                      # Flask application
├── docker-compose.yml          # Service definition (IaC)
├── Dockerfile                  # Container definition (IaC)
├── requirements.txt            # Python dependencies
└── test_app.py                 # Automated tests
```
