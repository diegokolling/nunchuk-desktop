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

### Code generation scripts

- `gen_qml_registry.py` — scans `Qml/` for `.qml` files and regenerates `features/generated_qml_registry.hpp` (runtime hash lookup) and `features/generated_qml_keys.hpp` (compile-time namespace constants). Run after adding/renaming QML files.
- `generate_app_strings.py` — parses `localization/STR_QML.js` and generates `core/common/resources/AppStrings.h` with `DEFINE_STRING_PROPERTY` macros. Run after adding/changing UI strings.

## Architecture

### Layer Model

The UI uses a **hierarchical layer system** with four host containers defined in `main.qml`:

- **ScreenHost** (`Qml/core/ScreenHost.qml`) — full-screen views via a `Loader`, mutually exclusive
- **PopupHost** (`Qml/core/PopupHost.qml`) — modal dialogs, stacks up to 4 via `Repeater` + `ListModel`
- **SubScreenHost** (`Qml/core/SubScreenHost.qml`) — inline overlays with semi-transparent backdrop
- **ToastHost** (`Qml/core/ToastHost.qml`) — temporary auto-dismissing `Popup` notifications

### State Machine (Legacy Pattern)

The original navigation system in `QAppEngine/QEventProcessor/` uses a state machine. Each screen is an `APPLICATION_STATE` struct (defined in `QAppEngine/QEventProcessor/Common/QCommonStructs.h`):

```cpp
typedef struct st_application_states {
    uint id;                      // E::STATE_ID_SCR_* enum
    void (*funcEntry)(QVariant);  // Entry callback
    void (*funcExit)(QVariant);   // Exit callback
    LAYER layerbase;              // LAYER_SCREEN, LAYER_ONSCREEN, LAYER_POPUP, LAYER_TOAST
    LIMIT duration;               // NONE or SEC_4 (for toasts)
    QString QmlPath;              // qrc:/ path to QML file
} APPLICATION_STATE;
```

~95 screens are registered in `Views/Views.h`. Event/state ID enums live in `Views/Common/ViewsEnums.h` (class `E` with `Q_ENUMS`). QML path `#define`s live in `Views/Common/ViewsDefines.h` under `namespace LocalMode` and `namespace OnlineMode`.

Event flow: `QEventProcessor::sendEvent(eventID)` → check popup/screen triggers → execute callback or `handleTransition()` → load QML for new state.

**To add a screen in this pattern:**
1. Add enum to `Views/Common/ViewsEnums.h` (`STATE_ID_SCR_*` and per-screen `EVT_*` events)
2. Add QML path `#define` to `Views/Common/ViewsDefines.h`
3. Create entry/exit callbacks in `Views/STATE_ID_SCR_*.cpp`
4. Register the `APPLICATION_STATE` in `Views/Views.h`
5. Create QML file in `Qml/Screens/LocalMode/` or `OnlineMode/`

### Clean Architecture (Modern Pattern)

Newer features use a clean architecture in `core/` and `features/`:

**UseCases** (`core/usecase/`) — X-macro pattern generates input/output structs with JSON serialization:

```cpp
// In a feature's usecases/ header:
#define FIELDS_INPUT(X)  X(QString, wallet_id)
DEFINE_USECASE_INPUT(MyInput)       // generates struct with field + toJson()

#define FIELDS_RESULT(X) X(QJsonObject, data)
DEFINE_USECASE_RESULT(MyResult)     // generates struct with field + toJson()

class MyUseCase : public AsyncUseCase<MyUseCase, MyInput, MyResult> {
    Result<MyResult> execute(const MyInput &input) override;
};
#undef FIELDS_INPUT
#undef FIELDS_RESULT
```

`AsyncUseCase` (CRTP) runs `execute()` on a `WorkerConcurrent<T>` thread and delivers results via callback. Macros defined in `core/usecase/DefineUseCaseDataType.h`.

**ViewModels** (`core/viewmodel/`) — hierarchy: `BaseViewModel` → `ActionViewModel` → `LoadingViewModel`. Lifecycle hooks: `onInit()`, `onAppear()`, `onDispose()`. Slots: `cancel()`, `back()`, `next()`, `close()`. Access DI via `ctx()` which returns a `ViewModelContext` wrapping `AppContext`.

The `DEFINE_QT_PROPERTY(Type, Name)` macro in `core/viewmodel/DefinePropertyMacros.h` generates a private member, getter, setter with change detection, Q_PROPERTY declaration, and change signal — eliminating Qt property boilerplate.

**Flows** (`core/flow/`) — `BaseFlow` subclasses represent navigation sequences. Each must implement `id()`. UI flows are mutually exclusive (starting one stops the previous). Background flows (`isBackground() = true`) can run concurrently. Flows emit `finished(FlowResult)` where `FlowResult` is `{Next, Back, Complete, Fail, Custom}`.

**Orchestrators** (`core/orchestrator/`) — `FeatureOrchestrator` holds a `QVector<FlowRule>` transition table. When a flow finishes, it matches `{from, FlowResult, to}` to start the next flow. `AppOrchestrator` holds a `QMap<QString, FeatureOrchestrator*>` for all features.

### Feature Modules

Each feature in `features/` is self-contained with `flows/`, `usecases/`, `viewmodels/` subdirectories. Current features: `wallets`, `signers`, `claiming`, `inheritance`, `draftwallets`, `transactions`, `common`.

**To add a feature in the modern pattern:**
1. Create `features/myfeature/` with `flows/`, `usecases/`, `viewmodels/` subdirs
2. Define UseCases extending `AsyncUseCase<Derived, Input, Output>` using the X-macro pattern
3. Define ViewModels extending `BaseViewModel` (or `ActionViewModel`/`LoadingViewModel`)
4. Create `viewmodels/DefineViewModel.hpp` with a `registerViewModels()` function:
   ```cpp
   namespace features::myfeature::viewmodels {
   static inline void registerViewModels() {
       const char* uri = "Features.MyFeature.ViewModels";
       REGISTER_VIEWMODEL(MyViewModel)  // expands to qmlRegisterType
   }}
   ```
5. Add the call to `app/QmlRegister.cpp`
6. Create corresponding QML files; import as `import Features.MyFeature.ViewModels 1.0`

### C++ ↔ QML Bridge

**Context properties** (global QML objects): `AppModel`, `Draco`, `ClientController`, `AppSetting` — set via `QEventProcessor::setContextProperty()` in `app/main.cpp`. Accessed directly by name in any QML file.

**Type registration** in `app/main.cpp`: `qmlRegisterType<ENUNCHUCK>("NUNCHUCKTYPE", ...)`, `qmlRegisterType<E>("HMIEVENTS", ...)`, etc. The `ENUNCHUCK` class in `ifaces/bridgeifaces.h` bridges all nunchuk enums (AddressType, WalletType, Chain, TransactionStatus, HealthStatus, hardware signer types) via `Q_ENUMS`.

**Bridge namespace** (`ifaces/`): Free functions in `namespace bridge` wrap libnunchuk calls. Error handling uses `QWarningMessage &msg` as the last parameter — callers must check it after each call. Smart pointer typedefs: `QWalletPtr`, `QTransactionPtr`, `QMasterSignerPtr`, etc. (`OurSharedPointer<T>`). Specialized bridge files: `bridgeWallet.cpp`, `bridgeTransaction.cpp`, `bridgeSigner.cpp`.

**nunchukiface** (`ifaces/nunchuckiface.h`) — singleton wrapping `std::unique_ptr<nunchuk::Nunchuk>` (two instances: `[0]=LOCAL`, `[1]=ONLINE`). Provides listener registration: `AddBalanceListener`, `AddTransactionListener`, `AddDeviceListener`, `AddBlockListener`, `AddBlockchainConnectionListener`.

### Key Singletons

| Class | Location | Purpose |
|---|---|---|
| `AppModel` | `Models/AppModel.h` | Central app state: wallet/signer/transaction lists, fees, block height. All QML screens read from here. |
| `AppSetting` | `Models/AppSetting.h` | User preferences and configuration |
| `ClientController` | `Models/Chats/ClientController.h` | Matrix protocol chat, contacts, rooms |
| `Draco` | `ifaces/` | Server communication |
| `QEventProcessor` | `QAppEngine/` | State machine, QML engine, context property management |
| `AppContext` | `app/AppContext.h` | DI container: `flowManager()`, `screenManager()`, `popupManager()`, `toastManager()`, `subScreenManager()` |
| `nunchukiface` | `ifaces/nunchuckiface.h` | Singleton wrapping the core libnunchuk C++ library |

### Data Models

Models extend `QAbstractListModel` for QML ListView integration. Pattern: define a `Roles` enum, implement `roleNames()`, `rowCount()`, `data()`. Items stored as `QList<QSharedPtr<T>>` with `QReadWriteLock` for thread safety. Key models: `WalletListModel`, `MasterSignerListModel`, `SingleSignerListModel`, `DeviceListModel`, `TransactionListModel` (all in `Models/`).

### Threading

`WorkerConcurrent<Result>` in `core/threads/` wraps `QFuture`/`QFutureWatcher` for off-main-thread execution with callback. Used by `AsyncUseCase` to keep UI responsive during libnunchuk calls.

### Localization

Strings defined as `var STR_QML_NNN = qsTr("...")` in `localization/STR_QML.js`. C++ equivalents auto-generated into `core/common/resources/AppStrings.h`. Qt translation files: `localization/nunchuk_en.ts` / `.qm`.

### Naming Conventions

| Component | Pattern | Example |
|---|---|---|
| Screen QML paths | `SCR_*` | `SCR_HOME`, `SCR_ADD_WALLET` |
| State IDs | `E::STATE_ID_SCR_*` | `E::STATE_ID_SCR_HOME` |
| Events | `EVT_*` | `EVT_HOME_WALLET_SELECTED` |
| UseCase types | `*Input`, `*Result` | `FetchWalletListInput` |
| ViewModels | `*ViewModel` | `WalletListViewModel` |
| Flows | `*Flow` | `SyncWalletFromRemoteFlow` |
| UI strings | `STR_QML_*` | `STR_QML_000` |
| Feature namespaces | `features::<name>::<layer>` | `features::wallets::viewmodels` |

### Git Submodules

| Path | Repository | Purpose |
|---|---|---|
| `contrib/libnunchuk` | nunchuk-io/libnunchuk | Core Bitcoin/wallet C++ library |
| `contrib/quotient` | tongvanlinh/libQuotient | Matrix protocol client (chat) |
| `contrib/zxing` | zxing-cpp/zxing-cpp | QR code encoding/decoding |
