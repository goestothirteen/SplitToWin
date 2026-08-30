# Deploy and operate SplitToWin. Run from ~/splittowin.
SHELL := /bin/bash

.PHONY: help deploy pull build-ui build-api up down restart logs logs-api ps health rollback versions clean

help:  ## show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN{FS=":.*?## "}{printf "  \033[36m%-12s\033[0m %s\n", $$1, $$2}'

deploy: pull build-ui build-api up health  ## full deploy: pull, build both, restart, verify
	@echo "deployed — https://split2win.duckdns.org"

pull:  ## fetch latest code for both repos
	git -C api pull --ff-only
	git -C ui pull --ff-only

build-ui:  ## build the frontend into ./dist (served directly by Caddy)
	@# Build into a staging dir first, so a failed build never leaves a
	@# half-written dist/ being served to whoever is mid-dinner.
	rm -rf dist.new
	docker build --target dist --output type=local,dest=./dist.new ./ui
	@test -f dist.new/index.html || { echo "build produced no index.html - keeping the old dist"; rm -rf dist.new; exit 1; }
	@# Sync contents rather than swapping the directory: Caddy bind-mounts
	@# dist/, and a bind mount follows the inode, so replacing the directory
	@# would silently leave the proxy serving the old one. --delete clears
	@# stale fingerprinted assets from previous builds.
	@mkdir -p dist
	rsync -a --delete dist.new/ dist/
	rm -rf dist.new
	@echo "frontend updated"

build-api:  ## rebuild the API image
	docker compose build api

up:  ## start (or restart) the API
	docker compose up -d

down:  ## stop the API
	docker compose down

restart:  ## restart the API without rebuilding
	docker compose restart api

logs:  ## follow API logs
	docker compose logs -f --tail=100 api

health:  ## check the deployed site end to end
	@echo -n "api container: "; docker inspect --format '{{.State.Health.Status}}' splittowin-api 2>/dev/null || echo "not running"
	@echo -n "api via caddy: "; curl -fsS https://split2win.duckdns.org/api/healthz && echo
	@echo -n "frontend:      "; curl -fsS -o /dev/null -w "HTTP %{http_code}\n" https://split2win.duckdns.org/

ps:  ## what's running
	docker compose ps

versions:  ## what's deployed right now
	@echo -n "api: "; git -C api log --oneline -1
	@echo -n "ui:  "; git -C ui log --oneline -1

rollback:  ## roll back to the previous commit of both repos: make rollback
	@echo "rolling back to the previous commit in each repo"
	git -C api reset --hard HEAD~1
	git -C ui reset --hard HEAD~1
	$(MAKE) build-ui build-api up health

clean:  ## reclaim docker build cache
	docker builder prune -f
