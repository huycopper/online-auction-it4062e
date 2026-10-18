# Server Online Auction
#
#   make             build bản thường      -> build/server
#   make SANITIZE=1  build bản AddressSanitizer + UBSan -> build-asan/server
#   make clean       xóa cả hai thư mục build
#
# Mỗi file src/x.c được biên dịch thành một file .o. Header nằm trong include/.

CC      := gcc
CFLAGS  := -std=c11 -Wall -Wextra -pedantic -g -D_DEFAULT_SOURCE -Iinclude -MMD -MP
LDFLAGS :=
LDLIBS  := -lcrypt
BUILD   := build

ifeq ($(SANITIZE),1)
BUILD   := build-asan
CFLAGS  += -O0 -fsanitize=address,undefined -fno-omit-frame-pointer
LDFLAGS += -fsanitize=address,undefined
endif

SRC := $(wildcard src/*.c)
OBJ := $(SRC:src/%.c=$(BUILD)/%.o)
BIN := $(BUILD)/server

.PHONY: all clean

all: $(BIN)

$(BIN): $(OBJ)
	@test -n "$(OBJ)" || { echo "Chưa có file .c nào trong server/src/"; exit 1; }
	$(CC) $(LDFLAGS) -o $@ $^ $(LDLIBS)

$(BUILD)/%.o: src/%.c | $(BUILD)
	$(CC) $(CFLAGS) -c -o $@ $<

$(BUILD):
	mkdir -p $@

clean:
	rm -rf build build-asan

-include $(OBJ:.o=.d)
