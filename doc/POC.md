# POC: GitOps з Argo CD на k3d (AsciiArtify)

> **Дата:** 2025-11-13

## Мета
Розгорнути локальний кластер Kubernetes (**k3d**) і встановити **Argo CD** для GitOps-керування застосунками. Надати команді доступ до **Web UI** Argo CD.

---

## Передумови
- Встановлено: `docker`, `kubectl`, `k3d`  
- (Опційно) CLI: `argocd`  
- Репозиторій клонований як `AsciiArtify`

---

## Крок 1 — Кластер k3d з мапінгом порту LB
```bash
# створити локальний registry та кластер з мапінгом 8080->80 на loadbalancer
k3d registry create k3d-registry.localhost --port 5001 || true
k3d cluster create asciiartify-gitops   --servers 1 --agents 2   --registry-use k3d-k3d-registry.localhost:5001   --port "8080:80@loadbalancer"
kubectl get nodes -o wide
```
Посилання: k3d — exposing services (приклад мапінгу порту LB). citeturn0search17turn0search3turn0search8

---

## Крок 2 — Встановлення Argo CD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server --timeout=180s
```
Офіційна інструкція «Installation». citeturn0search0

---

## Крок 3 — Доступ до UI Argo CD

### Варіант A: Port-forward (найпростіше)
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# UI: https://localhost:8080
```
Документація: «Try Argo CD locally». citeturn0search4

### Варіант B: LoadBalancer через k3d LB (для постійного доступу)
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
# UI: https://localhost:8080
```
(Для k3d порт 8080 уже промаплено на loadbalancer з Кроку 1.) Приклади підходів описані в гайдах/доках. citeturn0search17turn0search18

---

## Крок 4 — Отримати початковий пароль адміністратора
```bash
# варіант через kubectl:
kubectl -n argocd get secret argocd-initial-admin-secret   -o jsonpath="{{.data.password}}" | base64 --decode && echo

# або через CLI:
argocd admin initial-password -n argocd
```
Офіційний розділ «Getting Started» / команда `argocd admin initial-password`. citeturn0search1turn0search16

- Логін: **admin**
- Пароль: значення з секрету `argocd-initial-admin-secret`

> **Примітка безпеки:** змініть пароль одразу після першого входу.

---

## Крок 5 — Перевірка GitOps: приклад «guestbook»

### Декларативно (рекомендовано)
У цьому репо вже додано маніфест `gitops/applications/guestbook.yaml` з офіційного прикладу:
```bash
# створити неймспейс для прикладу
kubectl apply -f gitops/namespaces/guestbook.yaml

# зареєструвати застосунок в Argo CD декларативно
kubectl apply -f gitops/applications/guestbook.yaml

# відкрити UI та натиснути Sync (або почекати авто-синк якщо ввімкнути)
```
- Репозиторій з прикладами: `argoproj/argocd-example-apps` (шлях `guestbook`). citeturn0search2
- У UI побачите Application **guestbook**, після **Sync** розгорнуться `Deployment`/`Service`.

### Доступ до guestbook
```bash
# варіант через port-forward
kubectl -n guestbook port-forward svc/guestbook-ui 8081:80
# UI: http://localhost:8081
```

---

## One‑liner сценарій для всієї PoC
```bash
./scripts/poc-argocd-k3d.sh
```
Скрипт: створює кластер, ставить Argo CD, патчить LB, створює Application «guestbook» і підказує як отримати пароль.

---

## Прибирання
```bash
k3d cluster delete asciiartify-gitops
k3d registry delete k3d-registry.localhost
```

---

## Довідки
- **Installation** / **Getting Started** / **Port-forward** — офіційна документація Argo CD. citeturn0search0turn0search1turn0search4
- **Example Apps** — офіційний репозиторій прикладів. citeturn0search2
- **k3d Exposing Services / Config file** — офіційні матеріали k3d. citeturn0search17turn0search13
