# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Nunchuk Desktop is a cross-platform Bitcoin wallet application (C++20 / Qt 5 / QML) with hardware wallet integration and multi-signature support. The core Bitcoin functionality lives in the `contrib/libnunchuk` submodule.

## Build Commands

### Prerequisites

- Qt 5.12+ (tested with 5.15.2), GCC 14+ (Linux), Xcode (macOS), Visual Studio 2017+ (Windows)
- Git submodules: `git submodule update --init --recursive`
- Dependencies: libssl-dev, libsecret-1-dev, libboost-all-dev, Qt5Keychain, Berkeley DB 4.8, Olm

### Building libnunchuk dependencies first

```bash
# Bitcoin Core
cd contrib/libnunchuk/contrib/bitcoin
./autogen.sh
./configure --without-gui --disable-zmq --with-miniupnpc=no --with-incompatible-bdb --disable-bench --disable-tests -enable-module-ecdh
make -j$(nproc)

# SQLCipher
cd contrib/libnunchuk/contrib/sqlcipher
./configure --enable-tempstore=yes CFLAGS="-DSQLITE_HAS_CODEC" LDFLAGS="-lcrypto"
make -j$(nproc)
```

### Configure and build

```bash
# Linux (set compiler env first)
export CC=gcc-14 CXX=g++-14 RANLIB=gcc-ranlib-14 AR=gcc-ar-14 NM=gcc-nm-14

cmake -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_PREFIX_PATH=~/Qt/5.15.2/gcc_64/lib/cmake \
  -DQt5_DIR=~/Qt/5.15.2/gcc_64/lib/cmake/Qt5
cmake --build build -j$(nproc)
```

### CMake options

| Option | Default | Description |
|---|---|---|
| `USING_WEBENGINE` | ON | Enable Qt WebEngine for OAuth sign-in |
| `LIMIT_SW_SIGNER` | OFF | Limit software signer functionality |
| `USING_AUTO_FEE` | ON | Enable automatic fee selection |

### Environment variables

`OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `OAUTH_REDIRECT_URI` — compiled into the binary at build time. Fall back to empty string if unset.

### No test suite or linter

There is no unit test framework or linting configuration in this project. The compiler flag `-Wall` is enabled, and `-Werror=return-type` is enforced on non-Windows platforms.

## Architecture

### Layer Model

The UI uses a **hierarchical layer system** with four host containers defined in `main.qml`:

- **ScreenHost** — full-screen views (mutually exclusive, each replaces the previous)
- **PopupHost** — modal dialogs (stack up to 4)
- **SubScreenHost** — inline overlay screens
- **ToastHost** — temporary auto-dismissing notifications

### State Machine (Legacy Pattern)

The original navigation system in `QAppEngine/QEventProcessor/` uses a state machine where each screen is an `APPLICATION_STATE` struct with an ID, entry/exit callbacks, layer type, and a QML path. ~95 screens are registered in `Views/Views.h`.

Event flow: `QEventProcessor::sendEvent(eventID)` → check popup/screen triggers → execute callback or `handleTransition()` → load QML for new state.

To add a screen in this pattern:
1. Define entry/exit functions in `Views/STATE_ID_SCR_*.cpp`
2. Register the `APPLICATION_STATE` in `Views/Views.h`
3. Create the QML file in `Qml/Screens/`

### Clean Architecture (Modern Pattern)

Newer features use a clean architecture in `core/` and `features/`:

- **UseCases** (`core/usecase/`) — business logic with macro-generated input/output types via `DEFINE_USECASE_INPUT` / `DEFINE_USECASE_RESULT`. Extend `AsyncUseCase<Derived, Input, Output>` and implement `execute()`.
- **ViewModels** (`core/viewmodel/`) — extend `BaseViewModel` with lifecycle hooks: `onInit()`, `onAppear()`, `onDispose()`, plus slots `cancel()`, `back()`, `next()`, `close()`.
- **Flows** (`core/flow/`) — extend `BaseFlow` for multi-screen navigation sequences. `FlowManager` tracks active flows. `FlowRule` structs define state transitions.
- **Orchestrators** (`core/orchestrator/`) — `FeatureOrchestrator` wires flows together using `FlowRule` transition tables.

### Feature Modules

Each feature in `features/` is self-contained:

```
features/<name>/
├── flows/        # BaseFlow subclasses
├── usecases/     # AsyncUseCase implementations
└── viewmodels/   # BaseViewModel subclasses + DefineViewModel.hpp
```

ViewModels are registered in each feature's `DefineViewModel.hpp` using `REGISTER_VIEWMODEL` macros. All features are registered centrally in `app/QmlRegister.cpp`.

### C++ ↔ QML Bridge

- **Singletons as context properties**: `AppModel`, `Draco` (server comms), `ClientController` are set via `QEventProcessor::setContextProperty()` and accessible globally in QML.
- **Type registration**: C++ types registered with `qmlRegisterType<>()` in `app/main.cpp` (e.g., `ENUNCHUCK`, `EVT`, `QRCodeItem`, `DashRectangle`).
- **Bridge files** in `ifaces/`: `bridgeifaces.cpp` (~150KB) wraps `libnunchuk` for the frontend. Specialized bridges exist for wallets, transactions, and signers.

### Key Singletons

| Class | Location | Purpose |
|---|---|---|
| `AppModel` | `Models/AppModel.h` | Central app state: wallets, signers, transactions |
| `AppSetting` | `Models/AppSetting.h` | User preferences and configuration |
| `Draco` | `ifaces/` | Server/chat communication (Matrix protocol) |
| `QEventProcessor` | `QAppEngine/` | State machine and QML engine management |
| `AppContext` | `app/AppContext.h` | DI container: FlowManager, ScreenManager, PopupManager, ToastManager |

### QML Structure

```
Qml/
├── Screens/
│   ├── LocalMode/     # Local-only wallet screens
│   └── OnlineMode/    # Cloud-connected screens (accounts, groups)
├── Components/        # Reusable UI components
├── Popups/            # Modal dialog definitions
├── Global/            # Global QML singletons and utilities
└── core/              # ScreenHost, PopupHost, SubScreenHost, ToastHost
```

### Git Submodules

| Path | Repository | Purpose |
|---|---|---|
| `contrib/libnunchuk` | nunchuk-io/libnunchuk | Core Bitcoin/wallet C++ library |
| `contrib/quotient` | tongvanlinh/libQuotient | Matrix protocol client (chat) |
| `contrib/zxing` | zxing-cpp/zxing-cpp | QR code encoding/decoding |

### Dual UI Modes

The app supports **LocalMode** (offline wallet management) and **OnlineMode** (cloud sync, groups, premium features). These correspond to separate QML screen hierarchies under `Qml/Screens/`.
