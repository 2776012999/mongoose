CFLAGS  = -W -Wall -Wextra -Werror -Wundef -Wshadow -Wdouble-promotion
CFLAGS += -Wformat-truncation -fno-common -Wconversion -Wno-sign-conversion
CFLAGS += -g3 -Os -ffunction-sections -fdata-sections
CFLAGS += -I. -Icmsis_core/CMSIS/Core/Include -Icmsis_f4/Include
CFLAGS += -mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16
LDFLAGS ?= -Tlink.ld -nostdlib -nostartfiles --specs nano.specs -lc -lgcc -Wl,--gc-sections -Wl,-Map=$@.map

SOURCES = main.c syscalls.c sysinit.c
SOURCES += cmsis_f4/Source/Templates/gcc/startup_stm32f429xx.s # ST startup file. Compiler-dependent!

# Mongoose-specific source code files and build options. See https://mongoose.ws/documentation/#build-options
SOURCES += mongoose.c net.c packed_fs.c
CFLAGS += -DMG_ENABLE_TCPIP=1 -DMG_ARCH=MG_ARCH_NEWLIB -DMG_ENABLE_CUSTOM_MILLIS=1
CFLAGS += -DMG_ENABLE_CUSTOM_RANDOM=1 -DMG_ENABLE_PACKED_FS=1 -DMG_ENABLE_DRIVER_STM32=1 $(CFLAGS_EXTRA)

# Example specific build options. See README.md
CFLAGS += -DHTTP_URL=\"http://0.0.0.0/\"

ifeq ($(OS),Windows_NT)
  RM = cmd /C del /Q /F /S
else
  RM = rm -rf
endif

all build example: firmware.bin

firmware.bin: firmware.elf
	arm-none-eabi-objcopy -O binary $< $@

firmware.elf: cmsis_core cmsis_f4 $(SOURCES) hal.h link.ld
	arm-none-eabi-gcc $(SOURCES) $(CFLAGS) $(LDFLAGS) -o $@

flash: firmware.bin
	st-flash --reset write $< 0x8000000

cmsis_core:     # ARM CMSIS core headers
	git clone --depth 1 -b 5.9.0 https://github.com/ARM-software/CMSIS_5 $@
cmsis_f4:       # ST CMSIS headers for STM32F4 series
	git clone --depth 1 -b v2.6.8 https://github.com/STMicroelectronics/cmsis_device_f4 $@

# Automated remote test. Requires env variable VCON_API_KEY set. See https://vcon.io/automated-firmware-tests/
DEVICE_URL ?= https://dash.vcon.io/api/v3/devices/2
update: firmware.bin
	curl --fail-with-body -su :$(VCON_API_KEY) $(DEVICE_URL)/ota --data-binary @$<

test update: CFLAGS_EXTRA += -DUART_DEBUG=USART1
test: update
	curl --fail-with-body -su :$(VCON_API_KEY) $(DEVICE_URL)/tx?t=5 | tee /tmp/output.txt
	grep 'READY, IP:' /tmp/output.txt       # Check for network init
#	grep 'MQTT connected' /tmp/output.txt   # Check for MQTT connection success

clean:
	$(RM) firmware.* *.su cmsis_core cmsis_f4
