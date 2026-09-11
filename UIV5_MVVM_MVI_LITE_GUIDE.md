# Hướng dẫn triển khai feature UIV5 theo MVVM + MVI-lite

Tài liệu này là quy ước chung để triển khai các màn hình UIV5 mới có cấu trúc đồng nhất với `deliveryInfo`.

## 1. Kiến trúc và luồng dữ liệu

```text
User interaction
    -> Fragment tạo Action
    -> ViewModel.onAction(action)
    -> ViewModel xử lý nghiệp vụ / gọi DataSource
    -> ViewModel cập nhật một UiState duy nhất
    -> Fragment quan sát và render(state)

Sự kiện chỉ xảy ra một lần:
ViewModel -> Effect -> Fragment điều hướng / hiển thị thông báo
```

Quy ước cốt lõi:

- Fragment chỉ xử lý View, listener, render và điều hướng.
- Mọi ý định của người dùng đi vào ViewModel qua `onAction()`.
- ViewModel là nơi điều phối nghiệp vụ và thay đổi state.
- Màn hình có một `UiState` duy nhất, được public dưới dạng `StateFlow` chỉ đọc.
- Điều hướng, toast, dialog hoặc kết quả gửi sang màn khác dùng `Effect`.
- Validation và chuẩn hóa dữ liệu dùng Kotlin thuần.
- ViewModel không gọi Retrofit/service trực tiếp; dữ liệu đi qua interface DataSource/Repository.

## 2. Cấu trúc thư mục đề xuất

```text
featureName/
├── Uiv5FeatureNameFragment.kt
├── model/
│   ├── FeatureNameContract.kt       # UiState, Action, Effect, Message
│   ├── FeatureNameModels.kt         # UI model, draft, enum
│   ├── FeatureNameValidator.kt      # Kotlin thuần
│   └── FeatureNameDataSource.kt     # Interface + implementation truy cập repo
└── viewmodel/
    └── Uiv5FeatureNameViewModel.kt

app/src/test/.../featureName/
└── Uiv5FeatureNameViewModelTest.kt
```

Nếu feature lớn, tách `data/`, `domain/`, `ui/`. Không tạo nhiều tầng chỉ để đúng hình thức cho một màn hình nhỏ.

Layout của Custom View/Widget UIV5 phải theo quy tắc:

```text
uiv5_<tên_widget_snake_case>_view.xml
```

## 3. Contract chuẩn

```kotlin
data class FeatureUiState(
    val data: FeatureUiModel = FeatureUiModel(),
    val loading: Boolean = false,
    val error: FeatureValidationError? = null,
    val isSubmitEnabled: Boolean = false
)

sealed interface FeatureAction {
    data class Initialize(val args: FeatureArgs?) : FeatureAction
    data class InputChanged(val value: String) : FeatureAction
    data object SubmitClicked : FeatureAction
    data object BackClicked : FeatureAction
}

sealed interface FeatureEffect {
    data object NavigateBack : FeatureEffect
    data class ShowMessage(val message: FeatureMessage) : FeatureEffect
    data class SubmitSuccess(val result: FeatureResult) : FeatureEffect
}
```

Phân loại dữ liệu:

| Loại | Dùng cho | Ví dụ |
|---|---|---|
| `UiState` | Trạng thái bền vững để render lại | text, loading, lỗi field, enable button |
| `Action` | Ý định/thao tác đầu vào | click, nhập text, chọn địa chỉ |
| `Effect` | Sự kiện tiêu thụ một lần | navigate, toast, dialog, trả kết quả |

Không đặt điều hướng hoặc toast trong `UiState`, vì chúng có thể chạy lại khi Fragment được tạo lại.

## 4. ViewModel chuẩn

```kotlin
class Uiv5FeatureViewModel(
    private val dataSource: FeatureDataSource = RepositoryFeatureDataSource()
) : MyVNPTViewModel() {

    private val _uiState = MutableStateFlow(FeatureUiState())
    val uiState: StateFlow<FeatureUiState> = _uiState.asStateFlow()

    private val _effects = MutableSharedFlow<FeatureEffect>(extraBufferCapacity = 1)
    val effects: SharedFlow<FeatureEffect> = _effects.asSharedFlow()

    fun onAction(action: FeatureAction) {
        when (action) {
            is FeatureAction.Initialize -> initialize(action.args)
            is FeatureAction.InputChanged -> updateInput(action.value)
            FeatureAction.SubmitClicked -> submit()
            FeatureAction.BackClicked ->
                _effects.tryEmit(FeatureEffect.NavigateBack)
        }
    }

    private fun updateInput(value: String) {
        _uiState.update { state ->
            state.copy(data = state.data.copy(input = value)).validated()
        }
    }

    private fun FeatureUiState.validated(): FeatureUiState {
        val error = FeatureValidator.validate(data.input)
        return copy(
            error = error,
            isSubmitEnabled = error == null
        )
    }
}
```

Quy tắc:

- Chỉ `_uiState` được phép mutable; bên ngoài chỉ thấy `StateFlow`.
- Cập nhật state bằng `update { old -> old.copy(...) }`.
- Sau mỗi thay đổi ảnh hưởng form, chạy lại cùng một hàm `validated()`.
- Không sửa View, lấy `Context` hoặc đọc string resource trong ViewModel.
- Không truyền response API thẳng ra UI; map thành UI model hoặc domain model cần thiết.
- Dependency phải là interface để unit test có thể truyền fake.

## 5. Fragment chuẩn

```kotlin
private val viewModel: Uiv5FeatureViewModel by viewModels()
private var isRenderingState = false

override fun onInitView() {
    setupListeners()
    viewModel.onAction(FeatureAction.Initialize(readArgs()))
}

override fun onObserverViewModel() {
    observerStateFlow(viewModel.uiState, Lifecycle.State.STARTED, ::renderState)

    viewLifecycleOwner.lifecycleScope.launch {
        viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            viewModel.effects.collect(::handleEffect)
        }
    }
}

private fun setupListeners() {
    binding.editInput.afterTextChanged {
        if (isRenderingState) return@afterTextChanged
        viewModel.onAction(FeatureAction.InputChanged(it.toString()))
    }
}

private fun renderState(state: FeatureUiState) {
    isRenderingState = true
    try {
        if (binding.editInput.text != state.data.input) {
            binding.editInput.setText(state.data.input)
        }
    } finally {
        isRenderingState = false
    }

    binding.btnSubmit.isEnabled = state.isSubmitEnabled
}
```

`isRenderingState` ngăn `setText()` khi render kích hoạt `afterTextChanged` rồi gửi Action ngược lại ViewModel. Chỉ dùng cờ này cho cập nhật phát sinh từ state; thao tác thật của người dùng vẫn phải gửi Action.

Những logic được phép ở Fragment:

- `InputFilter`, vị trí con trỏ và thao tác trực tiếp với View.
- Chuyển validation enum thành string resource.
- Render loading/error/data.
- Xử lý Effect để navigate, toast hoặc dialog.

Không đặt quyết định nghiệp vụ, gọi API hoặc tự tính trạng thái nút Submit trong Fragment.

## 6. Validator Kotlin thuần

```kotlin
enum class FeatureValidationError {
    INPUT_REQUIRED,
    INPUT_INVALID
}

object FeatureValidator {
    fun sanitize(input: String): String = input.trim().take(255)

    fun validate(input: String): FeatureValidationError? = when {
        input.isBlank() -> FeatureValidationError.INPUT_REQUIRED
        else -> null
    }
}
```

Validator không được phụ thuộc `Context`, `View`, resource, API hoặc storage. Nó nhận input và trả kết quả xác định để có thể unit test trực tiếp.

## 7. DataSource và Repository

```kotlin
interface FeatureDataSource {
    suspend fun getData(): FeatureData
}

class RepositoryFeatureDataSource : FeatureDataSource {
    private val commonRepo = RepositoryFactory.getCommonRepo()

    override suspend fun getData(): FeatureData {
        val response = commonRepo.getFeatureData(/* request */)
        return response.toFeatureData()
    }
}
```

Quy tắc dự án:

- Luôn lấy repository qua `RepositoryFactory`; không tự khởi tạo repository/service.
- ViewModel chỉ phụ thuộc interface `FeatureDataSource` hoặc repository interface.
- Implementation chịu trách nhiệm tạo request, gọi repository, đọc cache/preference khi cần và map response.
- Nếu nhiều màn hình cùng dùng nghiệp vụ hoặc feature phát triển lớn, nâng abstraction thành `FeatureRepository` và đặt ở tầng data/domain phù hợp.
- Khi liên quan session/auth, kiểm tra cả `AppVNPT` và `SharePref`.

## 8. Unit test tối thiểu

Mỗi ViewModel nên kiểm tra:

- `Initialize` tạo state ban đầu đúng.
- Mỗi Action cập nhật state đúng.
- Validation và trạng thái enable nút đúng.
- Response bất đồng bộ đến muộn không ghi đè dữ liệu người dùng đã sửa.
- Loading thành công/thất bại cập nhật state và effect đúng.
- Effect Submit/Navigation chỉ phát khi đủ điều kiện.
- Fake DataSource được truyền qua constructor; test không gọi mạng hoặc `SharePref` thật.

Khi đổi tên hoặc cấu trúc field trong `UiState`, phải cập nhật test trong cùng thay đổi.

## 9. Checklist cho task mới

- [ ] Có đúng một `UiState` public dưới dạng `StateFlow` chỉ đọc.
- [ ] Mọi thao tác người dùng được biểu diễn bằng `Action` và đi qua `onAction()`.
- [ ] Navigate/toast/dialog/kết quả một lần được biểu diễn bằng `Effect`.
- [ ] Fragment chỉ listener, render và xử lý Effect.
- [ ] ViewModel chứa điều phối nghiệp vụ và không phụ thuộc Android View/Context.
- [ ] Validator là Kotlin thuần, dùng enum/error model thay vì string resource.
- [ ] API/storage được tách sau interface DataSource/Repository.
- [ ] Repository được lấy qua `RepositoryFactory`.
- [ ] State được cập nhật bằng `copy`, không sửa object mutable bên trong.
- [ ] Có cơ chế tránh vòng lặp `render -> setText -> listener -> Action`.
- [ ] Có test cho state transition, validation, async race và effect quan trọng.
- [ ] Custom View/Widget XML tuân thủ tên `uiv5_*_view.xml`.

## 10. Chỉ dẫn dành cho API/AI khi triển khai

Khi được yêu cầu tạo một feature UIV5 tương tự, hãy dùng tài liệu này làm mặc định và giữ thay đổi trong architectural slice hiện tại. Trước khi viết code:

1. Đọc feature gần nhất có cùng loại luồng.
2. Xác định rõ dữ liệu nào thuộc `UiState`, `Action` và `Effect`.
3. Xác định nguồn dữ liệu hiện có trong `RepositoryFactory`.
4. Tạo interface dữ liệu để ViewModel có thể test độc lập.
5. Triển khai validator Kotlin thuần.
6. Viết ViewModel state transition trước, sau đó nối Fragment để render.
7. Thêm hoặc cập nhật unit test cùng task.

Không tự ý chuyển một màn hình legacy sang kiến trúc này nếu task không yêu cầu. Với code mới trong UIV5, ưu tiên cấu trúc trên để các feature có cùng cách đọc, kiểm thử và mở rộng.
