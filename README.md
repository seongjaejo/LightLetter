# LightLetter — CNN 가속기 기반 손글씨 인식 및 BFSK 광통신 시스템

손글씨를 카메라로 촬영해 FPGA의 CNN 가속기로 인식하고, 인식한 문자를 BFSK 레이저 광통신으로 전송해
수신부에서 FFT로 복원하는 7인 팀 프로젝트입니다. Zybo Z7-20 두 대를 송신부와 수신부로 구성했습니다.

- **진행**: 경기도팹리스아카데미 온디바이스AI 시스템반도체 설계 2기 (2026.09.09 ~ 2026.10.15)
- **하드웨어**: Zybo Z7-20 ×2, Pcam 5C, 레이저 모듈, BPW34 광센서
- **개발 환경**: Vivado / Vitis 2020.2, Verilog, C, Python

## 시스템 구성

- **송신부**: 카메라 → VDMA → DDR → PS 글자 영역 추출(ArUco) → DMA → PL 이미지 전처리 → CNN → 패킷·CRC 생성 → BFSK 레이저 송신
- **수신부**: 광센서 → XADC → FFT Buffer → 128-point FFT → BFSK 복조 → CRC 검사 → UART → PC UI

## 본인 역할 (조성재, 팀장)

- 팀장으로서 팀원 간 의견 조율
- **이미지 전처리 RTL** 설계 및 개선 — 흑백 변환·극성 반전, 임계값 배경 제거, Bounding Box 기반 글자 추출, 종횡비 유지 정규화, 28×28 중앙 배치 (`tx/preprocess/`)
- **Convolution 연산부** 설계 및 검증 — 3채널 Line Buffer, 3×3 Window, 병렬 MAC, Adder Tree, Conv2 두 Phase 연산을 위한 연산부 재사용 구조 (`tx/cnn/`)
- **DDR–DMA–전처리–CNN 연동** 및 Vivado ILA를 이용한 실보드 프레임 캡처 확인

CNN 제어·Pooling·FC, BFSK 송수신부, FFT는 팀원들과 역할을 나누어 구현했습니다.

## 결과

- 'HELLO' 다섯 글자를 촬영·인식한 뒤 광통신으로 전송하고, 수신부에서 같은 문자열을 복원하는 시연 완료
- 전처리 및 CNN 가속기(PL 100 MHz)는 CPU(667 MHz) 대비 처리 시간 **6.22배 단축**, 에너지 **93.2% 감소** (프로젝트 측정 기준)

---

## How to run

Requires Vivado and Vitis 2020.2 on Windows. Run each board from its own folder.

### RX board

1. `rx\CREATE_VIVADO.cmd` creates `rx\vivado\project\rx_final.xpr`.
2. `rx\CREATE_VITIS.cmd` creates `rx\vitis\workspace` from `rx\vivado\export\FFT_RX_FINAL.xsa` and builds the `BFSK_RX_UART` app.
3. Open `rx\vitis\workspace` in Vitis and run `BFSK_RX_UART` on the board. Prebuilt files are in `rx\prebuilt\`.
4. `rx\START_UI.cmd` starts the PC UI at `http://127.0.0.1:8765`. Close any serial terminal that holds the board's COM port first.

### TX board

1. `tx\CREATE_VIVADO.cmd` creates `tx\vivado\project\tx_top.xpr`. Generate the bitstream, and export the XSA to `tx\vivado\export\` when the hardware changes.
2. Export the hardware (including bitstream) to `tx\vivado\export\tx_top_wrapper.xsa`, then run `tx\CREATE_VITIS.cmd`. It creates `tx\vitis\workspace`, generates the platform/BSP and builds `tx_fpga`. Running it again updates the platform from the same XSA and rebuilds. Close Vitis if it is using this workspace before running the script.
3. Open `tx\vitis\workspace` in Vitis and run `tx_fpga` on the board. The app links the original `tx\vitis\tx_fpga\src` sources, so subsequent source edits are included when rebuilt. The launcher uses `C:\Xilinx\Vitis\2020.2\bin\xsct.bat`; adjust `BFSK_XSCT` if Vitis is installed elsewhere.
4. `tx\START_UI.cmd` starts the capture-board viewer. Install its packages once with `pip install -r tx\ui\requirements.txt`.

## BFSK transmitter

The BFSK TX history is imported from Critical-mankind/BFSK_Tx.
BFSK TX RTL and testbenches: `tx/bfsk_tx/`. Nucleo receiver test: `tx/bfsk_tx/nucleo_checker/`.
The standalone TX test application and Vivado test projects are kept in `legacy/`.

## FPGA BFSK Optical Communication

FPGA-based optical communication project using BFSK modulation and 128-point FFT frequency detection.

## Current baseline

- Modulation: BFSK
- Bit 0: 60.096 kHz / FFT Bin 8
- Bit 1: 97.656 kHz / FFT Bin 13
- SYNC: 120.192 kHz / FFT Bin 16
- XADC Sampling Rate: 961.538 kSample/s
- FFT: 128-point
- Symbol: 256 samples / 266.24 us
- Preamble: 4 SYNC symbols
- SFD: 0xD5
- CRC: CRC-8, polynomial 0x07
- RTL language: Verilog

## Directory structure

The two Zybo Z7-20 boards each have their own tree. Inside a board tree, each
function block keeps its RTL, testbenches and models together, and `vivado/` and
`vitis/` hold the board-level Vivado project scripts and Vitis workspace.

```text
tx/                 transmitter board: camera -> ArUco -> preprocess -> CNN -> BFSK TX
  cnn/              rtl/, ip/ (packaged cnn_ip), golden/ (C), model/ (Python), tb/
  preprocess/       img_preprocess RTL and testbenches
  aruco/            notebook, calibration, marker sheets, host test harness
  bfsk_tx/          BFSK TX RTL, testbenches, Nucleo checker firmware
  camera/           Digilent P-CAM / HDMI IP repository
  ui/               capture-board viewer (START_UI.cmd)
  vivado/           create_project.tcl, block design Tcl, constraints/, export/ (XSA)
  vitis/            Vitis workspace: tx_fpga, tx_fpga_system, tx_top_wrapper
rx/                 receiver board: XADC -> FFT -> BFSK RX -> UART -> PC UI
  fft/  bfsk_rx/  snapshot/  ui/
  vivado/  vitis/  prebuilt/  tools/  docs/
legacy/             unused or superseded sources, kept under their old paths
```

Each board tree has `CREATE_VIVADO.cmd` and `CREATE_VITIS.cmd` to rebuild the
project from scripts with paths relative to the script, so a fresh clone works on any PC.

## Development rules

- RTL is written in Verilog.
- Do not delete previously PASSed code without a recorded reason.
- Keep PASSed testbenches for regression.

## RX UART to PC UI

The September 2026 RX build sends decoded packets to the PC through PS7 UART1 as CRLF-terminated CSV at 115200 baud. See `rx/docs/rx_uart_protocol.md` for the wire format and `rx/vitis/src/` for the Vitis application source.
