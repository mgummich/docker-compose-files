SECRETS_DIR := /dev/shm/traefik-secrets
SOPS_FILE   := .env.sops
COMPOSE     := docker compose -f compose.yaml

.PHONY: secrets-init secrets-clean up down logs ps encrypt decrypt edit

secrets-init: ## Decrypt secrets to RAM-backed tmpfs — never touches disk
	@mkdir -p $(SECRETS_DIR) && chmod 700 $(SECRETS_DIR)
	@sops --decrypt --extract '["CF_DNS_API_TOKEN"]'               $(SOPS_FILE) > $(SECRETS_DIR)/cf_dns_api_token
	@sops --decrypt --extract '["AUTHELIA_JWT_SECRET"]'            $(SOPS_FILE) > $(SECRETS_DIR)/authelia_jwt_secret
	@sops --decrypt --extract '["AUTHELIA_SESSION_SECRET"]'        $(SOPS_FILE) > $(SECRETS_DIR)/authelia_session_secret
	@sops --decrypt --extract '["AUTHELIA_STORAGE_ENCRYPTION_KEY"]' $(SOPS_FILE) > $(SECRETS_DIR)/authelia_storage_key
	@chmod 600 $(SECRETS_DIR)/*
	@echo "Secrets written to $(SECRETS_DIR) (RAM only)"

secrets-clean: ## Wipe secrets from RAM
	@rm -rf $(SECRETS_DIR)
	@echo "Secrets wiped from $(SECRETS_DIR)"

up: secrets-init ## Start stack — secret values never appear in env or docker inspect
	@DOMAIN=$$(sops --decrypt --extract '["DOMAIN"]' $(SOPS_FILE)) \
	 ACME_EMAIL=$$(sops --decrypt --extract '["ACME_EMAIL"]' $(SOPS_FILE)) \
	 $(COMPOSE) up -d

down: ## Stop stack and wipe secrets from RAM
	$(COMPOSE) down
	$(MAKE) secrets-clean

logs: ## Tail logs
	$(COMPOSE) logs -f

ps: ## Show service status
	$(COMPOSE) ps

encrypt: ## Encrypt .env → .env.sops (then delete .env)
	sops --encrypt .env > $(SOPS_FILE)
	rm -f .env
	@echo "Encrypted. Commit $(SOPS_FILE), keep .env out of git."

decrypt: ## Decrypt .env.sops → .env (for inspection only)
	sops --decrypt $(SOPS_FILE) > .env
	@echo "WARNING: .env is plaintext on disk. Delete after use."

edit: ## Edit secrets in-place without decrypting to disk
	sops $(SOPS_FILE)
