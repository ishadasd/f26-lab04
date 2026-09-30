# Lab 4 Starter: Deploy lab04-service

`lab04-service` is a small HTTP service with two routes. `GET /api/health` returns
`{"status":"ok"}` and `GET /api/rooms` returns a fixed list of rooms. It is already
written, already tested, and already containerized.

You write no code in this lab. You deploy the service to AWS, break it, fix it, and
tear it down, and you record what happened in `DEPLOYMENT.md`.

## Local warm-up

Run these before you touch AWS. They exercise the same image the deploy runs.

```
cd service && mvn test && cd ..
docker build -t lab04-service .
docker run -d --rm --name lab04-service -p 8080:8080 lab04-service
curl localhost:8080/api/health
docker logs lab04-service
docker stop lab04-service
```

The container prints one line when it comes up, which `docker logs` shows you:

```
lab04-service listening on 8080
```

Remember that line. You will look for it again.

## Deploy

Follow the Lab 4 handout on the course page. It walks the three milestones (deploy and
explain what got created, deploy the broken variant and diagnose it, tear it down).

## Where things are

- Service code: `service/src/main/java/edu/cmu/cs214/lab04/ServiceMain.java`
- Tests: `service/src/test/java/edu/cmu/cs214/lab04/ServiceMainTest.java`
- Container build: `Dockerfile`, `run.sh`
- Infrastructure: `infra/template.yaml`, with `infra/params-healthy.json` and
  `infra/params-scenario2.json`
- Your writeup: `DEPLOYMENT.md`
- Setup: `SETUP.md`

## Continuous integration

CI is configured in `.github/workflows/ci.yml`. Every push runs the service tests and
builds the container image. GitHub disables workflows on a fresh fork, so enable
them from the Actions tab if it asks.

## Agent assistance

Tool/model used: OpenAI Codex desktop, GPT-6 Astra (`gpt-6-astra`), for setup,
template review, evidence preparation, and explanations. No agent transcript is
published in this lab fork.

## Preparation status (2026-09-30)

Local Docker build, four service tests, and an external-to-container local health
check passed; outputs are in `evidence/`. All three AWS milestones are recorded in
`DEPLOYMENT.md`: healthy external check, scenario-2 SSM diagnosis and healthy
replacement, then confirmed stack deletion and Learner Lab termination.
