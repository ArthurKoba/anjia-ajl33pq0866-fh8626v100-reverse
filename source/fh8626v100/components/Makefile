CC ?= cc
CFLAGS ?= -std=c11 -Wall -Wextra -Werror -O2 -pthread

.PHONY: check check-board check-control check-diagnostics check-media-contracts clean

check: check-board check-control check-diagnostics check-media-contracts

check-board:
	@set -eu; bin=$$(mktemp /tmp/fh8626-board-test.XXXXXX); trap 'rm -f "$$bin"' EXIT; \
	$(CC) $(CFLAGS) -Iboard board/tests/test_dualsensor_rollback.c board/fh8626_board_dualsensor.c -o "$$bin"; \
	"$$bin"

check-control:
	@set -eu; bin=$$(mktemp /tmp/fh8626-control-test.XXXXXX); trap 'rm -f "$$bin"' EXIT; \
	$(CC) $(CFLAGS) -Icontrol control/tests/test_scene_transaction.c -o "$$bin"; \
	"$$bin"

check-diagnostics:
	@set -eu; bin=$$(mktemp /tmp/fh8626-diagnostics-test.XXXXXX); trap 'rm -f "$$bin"' EXIT; \
	$(CC) $(CFLAGS) -Idiagnostics diagnostics/tests/test_log_rotation.c -o "$$bin"; \
	"$$bin"

check-media-contracts:
	$(MAKE) -C media_contracts/tests clean
	$(MAKE) -C media_contracts/tests check
	$(MAKE) -C media_contracts/tests clean

clean:
	$(MAKE) -C media_contracts/tests clean
