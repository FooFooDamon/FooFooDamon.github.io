<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# `ESP32`的黑佬窝

```
sudo apt-get install git wget flex bison gperf python3 python3-pip python3-venv cmake ninja-build ccache libffi-dev libssl-dev dfu-util libusb-1.0-0

export IDF_ROOT_DIR="$HOME/esp-idf"
export IDF_VERSION=5.4.1
export IDF_GITHUB_ASSETS="dl.espressif.cn/github_assets"
export IDF_TOOLS_PATH="${IDF_ROOT_DIR}/idf_tools-${IDF_VERSION}"

[ -e "${IDF_ROOT_DIR}" ] || mkdir "${IDF_ROOT_DIR}"
[ -e "${IDF_TOOLS_PATH}" ] || mkdir "${IDF_TOOLS_PATH}"

cd "${IDF_ROOT_DIR}" # || exit 1




time git clone -b v${IDF_VERSION} --depth=1 --recursive https://github.com/espressif/esp-idf.git "esp-idf-${IDF_VERSION}" \
    && cd "esp-idf-${IDF_VERSION}" # || exit 1
# Try this command if git clone timed out: git submodule update --depth=1 --recursive




# Modify tools/idf_tools.py:
#
# action_install_python_env()
#     ...
#     run_args = [virtualenv_python, '-m', 'pip', 'install', '-i', 'https://pypi.tuna.tsinghua.edu.cn/simple', '--no-cache-dir', '--timeout', '300', '--upgrade', 'pip']
#     ...
#     run_args = [virtualenv_python, '-m', 'pip', 'install', '-i', 'https://pypi.tuna.tsinghua.edu.cn/simple', '--no-cache-dir', '--timeout', '300', '--upgrade', 'setuptools']
#     ...
#     run_args = [virtualenv_python, '-m', 'pip', 'install', '-i', 'https://pypi.tuna.tsinghua.edu.cn/simple', '--no-cache-dir', '--timeout', '300', '--no-warn-script-location']




./install.sh esp32s3

alias idf_enable=". ~/esp-idf/esp-idf-${IDF_VERSION}/export.sh"

idf_enable

# Update IDF_VERSION in ~/.bashrc




cd "${IDF_ROOT_DIR}"/examples/get-started/blink

idf.py --list-targets

idf.py set-target esp32s3

idf.py menuconfig # Update items such as CONFIG_ESPTOOLPY_FLASHSIZE, APP_BUILD_TYPE_*, etc.

time idf.py build # Or: mkdir build && cd build && cmake .. && make -j $(nproc)

export ESPPORT=/dev/ttyACM1

# esptool.py -p ${ESPPORT} flash_id

time idf.py flash
# Or run app in RAM (Note: APP_BUILD_TYPE_RAM must be selected):
# time esptool.py --chip esp32s3 -p ${ESPPORT} -b 460800 --before=default_reset --after=hard_reset --no-stub load_ram build/blink.bin

idf.py monitor
```

