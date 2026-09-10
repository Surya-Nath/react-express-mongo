# React + Express + MongoDB on Docker, ECR, and one EC2

**Nginx · React (static) · Express · MongoDB · GitHub Actions · Jenkins · Terraform · Ansible**

Browser hits **port 80 only**. Nginx serves the SPA and proxies `/api/` to Express. MongoDB is unpublished. Public `GET /` and `GET /api/` both returned `200` on a single EC2. The instance was destroyed the same day.

[Repository](https://github.com/Surya-Nath/react-express-mongo)

---

## The problem this lab exists to show

If a React app is built with `http://localhost:3001/api`, that string is **baked into the JS bundle**. On a phone or a laptop opening the EC2 public IP, `localhost` is the phone — not the server. The UI loads; every API call dies.

Fix used here:

- Axios already calls **relative** `/api`
- Nginx on `:80` has `location /api/` → `proxy_pass http://backend:3000`
- The image does **not** contain the EC2 DNS name

Dev `package.json` `"proxy"` only works with `npm start`. Production Nginx is a different program. That is the whole week.

---

## Request path

```
Internet :80
   ├── /        Nginx  → React build/ (static)
   └── /api/    Nginx  → Express :3000
                    └── mongodb://mongo:27017/TodoApp
```

| Service | Image | Host port |
| --- | --- | --- |
| frontend | ECR `week4-ui` (Nginx + `build/`) | **80** |
| backend | ECR `week4-api` (Node, `USER node`) | closed |
| mongo | `mongo:4.2.0` | **27017 closed** |

Two Compose networks: frontend = nginx+api, backend = api+mongo. The UI container cannot reach Mongo.

---

## UI image (multi-stage)

```
FROM node:22-alpine AS build
  npm ci --include=dev
  npm run build          → /ui/build

FROM nginx:1.27-alpine
  COPY build → html
  COPY nginx.conf
  USER nginx
```

`react-scripts` lives in `devDependencies`. `--omit=dev` on the **build** stage breaks the compile. `--omit=dev` on the **API** image is correct.

The final UI image has no `node_modules` and no `npm start`.

---

## Layout

```
frontend/Dockerfile      multi-stage Node → Nginx
frontend/nginx.conf      / + /api/ proxy
backend/Dockerfile      node:22-alpine, npm ci --omit=dev
docker-compose.yml       local, publish 80 only
compose.prod.yaml        ECR image: tags
.github/workflows/       week4-ui + week4-api
Jenkinsfile              same two images
terraform/               1× EC2, SG 22/lab-ip + 80/world
ansible/                 Docker, ECR login, compose up
```

---

## Local

```bash
docker compose up -d --build
curl -sI localhost          # Nginx 200
curl -sI localhost/api/     # Express JSON 200
```

`docker run` the UI image **alone** fails on `host not found in upstream "backend"`. That is expected. Compose attaches the DNS name.

---

## CI → ECR

Push `main`:

```
413816840602.dkr.ecr.ap-south-1.amazonaws.com/week4-ui:$SHA
413816840602.dkr.ecr.ap-south-1.amazonaws.com/week4-api:$SHA
```

Mongo stays a Hub pull on the box. It is not built or pushed.

---

## EC2

1. Terraform: Ubuntu 22.04 `t3.small`, key pair, default VPC  
2. Ansible: install Docker, copy `compose.prod.yaml` only (SPA + API are **in the images**)  
3. `curl http://$PUBLIC_IP/` and `curl http://$PUBLIC_IP/api/`  
4. `terraform destroy` the same evening  

No bind-mount of `src/`. That was Week 3 (Laravel). Here the artifacts are baked.

---

## Answers a reviewer will ask

| Question | Answer in this repo |
| --- | --- |
| Why did `localhost` in the bundle fail on EC2? | The browser, not Docker, resolves `localhost`. |
| What did multi-stage remove? | Node toolchain and `node_modules`. Final PID 1 is Nginx. |
| Who may open 27017? | Only `backend` on the overlay. Not the public NIC. |
| Nginx `/api` vs CORS on Express? | Same origin (`:80`) — proxy beats CORS for the browser. Sample still sends `Access-Control-Allow-Origin: *`. |
| `npm start` vs Nginx `dist`/`build`? | `npm start` is the CRA dev server. Prod is static files. |

---

## Security

- `.env`, `*.pem`, `*.tfstate`, live `inventory.ini` are gitignored  
- 27017 / 3000 unpublished  
- Jenkins credential id `aws-ecr` is not a GitHub deploy key  
- Lab IAM user is disposable; rotate if you fork  

---

## Stack

Docker Compose · React · Nginx · Express · MongoDB 4.2 · GitHub Actions · Jenkins · Amazon ECR · Terraform · Ansible · default VPC
