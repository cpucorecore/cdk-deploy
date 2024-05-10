START_DB := docker compose up -d zkevm-state-db
START_PROVER := docker compose up -d zkevm-prover
START_NODE := docker compose up -d zkevm-node
START_BRIDGE := docker compose up -d zkevm-bridge-service
START_DAC := docker compose up -d zkevm-data-availability

STOP_ALL := docker compose stop

run:
	$(START_DB)
	sleep 5
	$(START_PROVER)
	sleep 1
	$(START_NODE)
	sleep 5
	$(START_BRIDGE)
	sleep 1
	$(START_DAC)

run-bridge:
	$(START_BRIDGE)

stop:
	$(STOP_ALL)
