DB_NAME 			?= shop
DB_USER 			?= culiops
COMPOSE 			?= docker compose
GPGPASS		 		?= .gpgpass
EXEC 					= $(COMPOSE) exec -T db
PSQL 					= $(EXEC) psql -U $(DB_USER) -d $(DB_NAME) -v ON_ERROR_STOP=1
DUMP 					= $(EXEC) pg_dump -U $(DB_USER) $(DB_NAME)
BACKUP_DIR 		= backups
RTO_IN_SECOND = 60

up:
	$(COMPOSE) up -d --wait
	@echo "waiting for postgres to be healthy..."
	@echo "✓ db healthy"

down:
	$(COMPOSE) down

ps:
	$(COMPOSE) ps

logs:
	$(COMPOSE) logs -f db

seed:
	$(PSQL) < sql/seed.sql

count:
	@$(PSQL) -tAc "SELECT count(*) FROM orders" | xargs echo "orders rows:"

backup:
	@mkdir -pv $(BACKUP_DIR)
	@$(DUMP) --format=c > $(BACKUP_DIR)/orders-$$(date +%d-%m-%Y-%H-%M-%S).dump

disaster:
	@echo "💥 dropping table orders..."
	@$(PSQL) -c "DROP TABLE IF EXISTS orders"
	@echo "✓ data gone."

restore:
	@if [ -z "$(FILE)" ]; then \
	  echo "✗ Pass a dump file: make restore FILE=path/to/file.dump" >&2; \
	  exit 1; \
	fi; \
	if [ ! -f "$(FILE)" ]; then \
	  echo "✗ File not found: $(FILE)" >&2; \
	  exit 1; \
	fi; \
	start_time=$$(date +%s); \
	$(EXEC) pg_restore -U $(DB_USER) -d $(DB_NAME) --clean --if-exists < "$(FILE)"; \
	end_time=$$(date +%s); \
	elapsed=$$((end_time - start_time)); \
	echo "⏱ Restore time: $${elapsed}s (RTO: $(RTO_IN_SECOND)s)"; \
	if [ "$$elapsed" -gt "$(RTO_IN_SECOND)" ]; then \
	  echo "✗ RTO exceeded: $${elapsed}s > $(RTO_IN_SECOND)s" >&2; \
	  exit 1; \
	else \
	  echo "✓ Within RTO ($${elapsed}s <= $(RTO_IN_SECOND)s)"; \
	fi

drill:
	@echo "── 1. backup ──";       	  $(MAKE) -s backup
	@latest=$$(ls -1t "$(BACKUP_DIR)"/orders-*.dump 2>/dev/null | head -n 1); \
	if [ -z "$$latest" ]; then \
	  echo "✗ No backup file found in $(BACKUP_DIR)" >&2; \
	  exit 1; \
	fi; \
	echo "── 2. orders count ──";   	$(MAKE) -s count; \
	echo "── 3. DISASTER ──";       	$(MAKE) -s disaster; \
	echo "── 4. restore (timed) ──";  $(MAKE) -s restore FILE="$$latest"; \
	echo "── 5. verify ──";        	  $(MAKE) -s check-data-quality; \
	echo "✅ drill done. Did 'restored rows' match 'orders count'? If yes, you have a backup. If no, you had faith."

check-data-quality:
	@$(PSQL) -tAc "SELECT count(*) FROM orders" | xargs echo "restored rows:"

init-encryption-at-rest:
	@if [ ! -f $(GPGPASS) ]; then \
		head -c 24 /dev/urandom | base64 > $(GPGPASS); chmod 600 $(GPGPASS); \
		echo "✓ wrote $(GPGPASS). ⚠️  In production the key must NOT sit next to the backup." \
	else echo "$(GPGPASS) already exists"; \
	fi

backup-encrypt:
	@mkdir -p $(BACKUP_DIR)
	@$(DUMP) | gzip | gpg --batch --yes --symmetric --cipher-algo AES256 \
		--passphrase-file $(GPGPASS) -o $(BACKUP_DIR)/orders.sql.gz.gpg
	@echo "✓ $(BACKUP_DIR)/orders.sql.gz.gpg ($$(du -h $(BACKUP_DIR)/orders.sql.gz.gpg | cut -f1))"

restore-decrypt:
	@$(MAKE) -s disaster; \
	start_time=$$(date +%s); \
	gpg --batch --yes --decrypt --passphrase-file $(GPGPASS) $(BACKUP_DIR)/orders.sql.gz.gpg \
		| gunzip | $(PSQL); \
	end_time=$$(date +%s); \
	elapsed=$$((end_time - start_time)); \
	echo "⏱ Restore time: $${elapsed}s (RTO: $(RTO_IN_SECOND)s)"; \
	if [ "$$elapsed" -gt "$(RTO_IN_SECOND)" ]; then \
	  echo "✗ RTO exceeded: $${elapsed}s > $(RTO_IN_SECOND)s" >&2; \
	  exit 1; \
	else \
	  echo "✓ Within RTO ($${elapsed}s <= $(RTO_IN_SECOND)s)"; \
	fi

clean:
	@bash clean.sh
