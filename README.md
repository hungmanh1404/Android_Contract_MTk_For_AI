# Quy chuẩn triển khai Android Feature theo MVVM kết hợp UDF, mang phong cách MVI nhẹ.

## Cách sử dụng tài liệu này

Đưa toàn bộ file này cho AI/API trước khi yêu cầu triển khai một tính năng Android mới.

Prompt ngắn khuyến nghị:

```text
Hãy đọc MVVM_FEATURE_IMPLEMENTATION_SPEC.md và tuân thủ nó như implementation contract.

Tính năng cần làm: <mô tả tính năng>
API/GraphQL contract: <đường dẫn hoặc nội dung contract>
Thiết kế UI: <đường dẫn XML/Figma/mô tả>
Yêu cầu nghiệp vụ: <các rule>

Hãy kiểm tra convention hiện có của dự án, triển khai đầy đủ, viết test và báo cáo các giả định.
Không thay đổi những phần ngoài phạm vi nếu không thực sự cần thiết.
```

Tài liệu này là quy chuẩn mặc định. Nếu dự án có convention hợp lý và khác tài liệu, AI phải giữ convention của dự án, đồng thời vẫn bảo đảm các nguyên tắc cốt lõi: một nguồn state, dữ liệu một chiều, dependency rõ ràng, lifecycle an toàn và có thể test.

---

## 1. Mục tiêu kiến trúc

Mọi feature phải hướng tới:

- Dễ đọc: nhìn cấu trúc file là biết trách nhiệm của từng lớp.
- Dễ bảo trì: thay API/UI không làm lan truyền thay đổi không cần thiết.
- Dễ test: business rule và ViewModel test được mà không cần Android View.
- An toàn lifecycle: không giữ View, Fragment, Activity hoặc Context trong ViewModel.
- Một nguồn dữ liệu: ViewModel sở hữu toàn bộ state của màn hình.
- Dữ liệu một chiều: View phát Action; ViewModel phát State/Effect.
- Immutable: state public không chứa collection mutable hoặc property `var`.
- Không coupling ẩn: tránh singleton event bus và dependency global.

Luồng chuẩn:

```text
User interaction
    -> View phát Action
    -> ViewModel xử lý Action
    -> UseCase/Repository
    -> ViewModel tạo UiState mới hoặc Effect
    -> View render UiState / xử lý Effect
```

---

## 2. Việc AI phải làm trước khi viết code

AI phải kiểm tra tối thiểu:

1. Cấu trúc module và package hiện có.
2. `AGENTS.md`, README, coding convention hoặc instruction cục bộ.
3. Dự án dùng XML Fragment, Activity hay Jetpack Compose.
4. Cơ chế DI: Koin, Hilt/Dagger hoặc manual injection.
5. Cơ chế async/state: Coroutine, Flow, LiveData, RxJava.
6. Repository, API client, GraphQL/REST DTO và error wrapper hiện có.
7. Base class, extension, design system và component có thể tái sử dụng.
8. Convention test và thư viện mocking/fake đang dùng.
9. Navigation và cách truyền argument/result.
10. Các file đang có thay đổi của người dùng; không ghi đè thay đổi ngoài phạm vi.

AI không được tạo thêm một framework kiến trúc mới nếu dự án đã có giải pháp tương đương. Chỉ bổ sung lớp abstraction khi nó làm trách nhiệm rõ hơn hoặc tăng khả năng test thực tế.

Nếu thiếu thông tin nhỏ, AI được phép đưa ra giả định hợp lý và ghi rõ trong phần bàn giao. Nếu thiếu lựa chọn có thể làm thay đổi lớn hành vi hoặc dữ liệu, phải hỏi người dùng trước.

---

## thiết kế giao diện. Quy tắc sử dụng resource khi thiết kế giao diện
- Khi thiết kế giao diện bằng XML hoặc Fragment, không được hardcode màu sắc, kích thước, khoảng cách, margin, padding, textSize, cornerRadius, v.v.

- Trước khi tạo resource mới, phải kiểm tra và ưu tiên tái sử dụng resource đã có trong ứng dụng.
- Màu sắc phải được khai báo và sử dụng từ:
app/src/main/res/values/colors.xml
- Kích thước và khoảng cách phải được khai báo và sử dụng từ:
app/src/main/res/values/dimens.xml
- Nếu chưa có giá trị phù hợp, hãy tạo resource mới với tên rõ ràng, đúng quy ước của dự án.
- Không khai báo trực tiếp các giá trị như #FFFFFF, 16dp, 14sp trong XML hoặc code Kotlin.

## 3. Cấu trúc feature mặc định

```text
feature/<feature_name>/
├── <Feature>Contract.kt
├── <Feature>ViewModel.kt
├── <Feature>Fragment.kt          # Khi dùng XML
├── <Feature>Screen.kt            # Khi dùng Compose
├── adapter/                       # Chỉ khi XML/RecyclerView cần
│   └── <Item>Adapter.kt
└── mapper/                        # Chỉ tạo khi mapping đủ phức tạp
    └── <Feature>UiMapper.kt

domain/<feature_name>/             # Tạo khi có business rule/use case thật sự
├── <Action>UseCase.kt
├── <Feature>Repository.kt
└── model/

data/<feature_name>/
├── <Feature>RepositoryImpl.kt
├── remote/
└── mapper/
```

Không bắt buộc chia đủ ba layer cho feature CRUD rất nhỏ. Trong trường hợp đó, ViewModel có thể gọi repository interface trực tiếp. Không tạo use case chỉ để chuyển tiếp một lời gọi mà không có business rule.

---

## 4. Contract của màn hình

Mỗi màn hình ưu tiên gom `UiState`, `Action` và `Effect` trong `<Feature>Contract.kt`.

### 4.1. UiState

`UiState` là snapshot đầy đủ để render màn hình tại một thời điểm.

```kotlin
data class ExampleUiState(
    val form: ExampleForm = ExampleForm(),
    val items: List<ExampleItemUi> = emptyList(),
    val isInitialLoading: Boolean = false,
    val isRefreshing: Boolean = false,
    val isSubmitting: Boolean = false,
    val validation: ExampleValidation = ExampleValidation()
) {
    val canSubmit: Boolean
        get() = !isSubmitting && validation.isValid
}

data class ExampleForm(
    val name: String = "",
    val selectedOption: ExampleOptionUi? = null
)
```

Quy tắc bắt buộc:

- Property dùng `val`.
- Collection public dùng `List`, `Set`, `Map`, không dùng `MutableList`.
- Không đặt `View`, `Drawable`, `Context`, binding hoặc callback vào state.
- Không lưu cùng một dữ liệu nghiệp vụ ở cả ViewModel và adapter/custom view.
- Giá trị có thể suy ra đơn giản nên là derived property như `canSubmit`.
- State loading phải có nghĩa rõ ràng; tránh một boolean loading dùng cho mọi request nếu UI cần phân biệt.

### 4.2. Action

Action mô tả ý định của người dùng hoặc lifecycle event, không mô tả chi tiết widget.

```kotlin
sealed interface ExampleAction {
    data object ScreenStarted : ExampleAction
    data object Refresh : ExampleAction
    data class NameChanged(val value: String) : ExampleAction
    data class OptionSelected(val optionId: String?) : ExampleAction
    data class RemoveItem(val itemId: String) : ExampleAction
    data object Submit : ExampleAction
}
```

Tên tốt:

- `ProjectSelected`
- `CommentChanged`
- `RetryClicked`
- `LoadMore`
- `Submit`

Tên cần tránh:

- `SetTextViewValue`
- `UpdateRecyclerView`
- `CallApiFromButton`
- `OnClickButton1`

View gọi một entry point thống nhất:

```kotlin
viewModel.onAction(ExampleAction.Submit)
```

### 4.3. Effect

Effect là yêu cầu UI chỉ xử lý một lần:

```kotlin
sealed interface ExampleEffect {
    data class ShowMessage(val message: UiText) : ExampleEffect
    data class NavigateToDetail(val itemId: String) : ExampleEffect
    data object CloseScreen : ExampleEffect
}
```

Effect phù hợp cho:

- Navigation.
- Toast/snackbar/popup.
- Mở file picker, camera hoặc permission dialog.
- Trả result cho màn hình trước.

Không dùng Effect cho dữ liệu cần tồn tại sau recreate như danh sách, selection, loading hoặc nội dung form. Những dữ liệu đó phải nằm trong `UiState`.

---

## 5. ViewModel chuẩn

Template mặc định:

```kotlin
class ExampleViewModel(
    private val repository: ExampleRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    private val _uiState = MutableStateFlow(ExampleUiState())
    val uiState: StateFlow<ExampleUiState> = _uiState.asStateFlow()

    private val _effect = MutableSharedFlow<ExampleEffect>(
        extraBufferCapacity = 1
    )
    val effect: SharedFlow<ExampleEffect> = _effect.asSharedFlow()

    fun onAction(action: ExampleAction) {
        when (action) {
            ExampleAction.ScreenStarted -> loadInitialData()
            ExampleAction.Refresh -> refresh()
            is ExampleAction.NameChanged -> updateName(action.value)
            is ExampleAction.OptionSelected -> selectOption(action.optionId)
            is ExampleAction.RemoveItem -> removeItem(action.itemId)
            ExampleAction.Submit -> submit()
        }
    }
}
```

Quy tắc:

- Dependency được constructor inject.
- Public output là read-only `StateFlow`/`SharedFlow`.
- Chỉ ViewModel cập nhật `_uiState` và phát `_effect`.
- Mỗi thay đổi state tạo instance mới bằng `copy()`.
- ViewModel không gọi trực tiếp method của View.
- ViewModel không nhận callback từ Fragment để trả kết quả API.
- ViewModel không giữ `Activity`, `Fragment`, binding hoặc Android View.
- Chỉ giữ application `Context` khi thật sự cần và inject rõ ràng; ưu tiên abstraction thay thế.
- Payload gửi API được tạo từ state/domain model, không gom từ nhiều widget tại thời điểm submit.

### Cập nhật state

```kotlin
private fun updateName(value: String) {
    _uiState.update { current ->
        current.copy(
            form = current.form.copy(name = value)
        )
    }
}
```

Không làm:

```kotlin
uiState.value.form.name = value
items.value?.add(newItem)
```

### Coroutine

- Dùng `viewModelScope` trong ViewModel.
- Dùng đúng một owner chịu trách nhiệm `launch`; tránh helper tự launch bên trong một coroutine khác.
- Không nuốt `CancellationException`.
- Khi bắt `Throwable`, phải rethrow `CancellationException`.
- Search-as-you-type phải hủy request cũ bằng `flatMapLatest`, `mapLatest` hoặc quản lý `Job`.
- Không để response của query cũ ghi đè query mới.
- Request song song chỉ dùng khi độc lập và thật sự giảm thời gian chờ.

Ví dụ bắt lỗi an toàn:

```kotlin
private suspend fun <T> safeRequest(block: suspend () -> T): Result<T> =
    try {
        Result.success(block())
    } catch (cancellation: CancellationException) {
        throw cancellation
    } catch (throwable: Throwable) {
        Result.failure(throwable)
    }
```

---

## 6. Repository và mapping

ViewModel ưu tiên phụ thuộc repository interface dành cho domain/feature:

```kotlin
interface ExampleRepository {
    suspend fun getItems(query: String, page: Int): Page<ExampleItem>
    suspend fun submit(command: SubmitExampleCommand)
}
```

Implementation chịu trách nhiệm:

- Gọi REST/GraphQL/database.
- Chuyển error wrapper của data layer thành error domain có nghĩa.
- Map DTO/GraphQL object sang domain model.
- Không expose Apollo/Retrofit/Room DTO lên UI nếu điều đó làm UI phụ thuộc data source.
- Phân biệt chính xác `null`, absent và chuỗi rỗng theo API contract.
- Không dùng display name làm ID.

Mapping nên là pure function:

```kotlin
internal fun ItemDto.toDomain(): ExampleItem = ExampleItem(
    id = requireNotNull(id),
    name = name.orEmpty()
)
```

Không tạo một repository tổng hợp khổng lồ cho toàn ứng dụng nếu có thể chia theo domain/feature hợp lý.

---

## 7. Validation và submit

Business validation phải test được trong ViewModel/domain mà không cần inflate View.

```kotlin
private fun validate(state: ExampleUiState, showErrors: Boolean) =
    ExampleValidation(
        showErrors = showErrors,
        isNameValid = state.form.name.isNotBlank(),
        isOptionValid = state.form.selectedOption != null
    )
```

Quy tắc:

- View có thể quyết định cách hiển thị lỗi.
- ViewModel/domain quyết định dữ liệu có hợp lệ để submit hay không.
- Chặn submit lặp bằng `isSubmitting` hoặc cơ chế tương đương.
- Trim/normalize dữ liệu tại boundary rõ ràng.
- Payload lấy từ một state snapshot nhất quán.
- Optional field null không được tự động biến thành `""` nếu API phân biệt hai giá trị.

---

## 8. Pagination và search

Mỗi nguồn paging có state riêng:

```kotlin
data class PagingState(
    val query: String = "",
    val page: Int = 0,
    val totalPages: Int = 1,
    val isLoading: Boolean = false
) {
    val canLoadMore: Boolean
        get() = !isLoading && page < totalPages
}
```

Quy tắc:

- Không dùng một biến pagination global cho nhiều danh sách.
- Search mới reset page và thay danh sách.
- Load-more nối danh sách immutable.
- Chống gọi load-more trùng khi request đang chạy.
- Loại duplicate bằng stable ID nếu backend có thể trả item lặp.
- Nếu dự án đã dùng Paging 3, ưu tiên Paging 3 thay vì tự viết pagination.
- Response phải được kiểm tra còn khớp query/owner hiện tại trước khi ghi state.

---

## 9. Fragment dùng XML

Fragment chỉ làm bốn việc:

1. Khởi tạo View và adapter.
2. Chuyển UI event thành Action.
3. Collect State/Effect theo lifecycle.
4. Render state và xử lý effect.

Template:

```kotlin
class ExampleFragment : Fragment() {

    private var _binding: FragmentExampleBinding? = null
    private val binding: FragmentExampleBinding
        get() = requireNotNull(_binding)

    private val viewModel: ExampleViewModel by viewModel()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        initView()
        initListeners()
        collectOutput()
    }

    private fun initListeners() = with(binding) {
        submitButton.setOnClickListener {
            viewModel.onAction(ExampleAction.Submit)
        }
    }

    private fun collectOutput() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                launch { viewModel.uiState.collect(::render) }
                launch { viewModel.effect.collect(::handleEffect) }
            }
        }
    }

    private fun render(state: ExampleUiState) = with(binding) {
        submitButton.isEnabled = state.canSubmit
        progressBar.isVisible = state.isSubmitting
        adapter.submitList(state.items)
    }

    override fun onDestroyView() {
        binding.recyclerView.adapter = null
        _binding = null
        super.onDestroyView()
    }
}
```

Không làm trong Fragment:

- Gọi API/repository trực tiếp.
- Tạo request payload từ nhiều widget.
- Giữ business state chính trong adapter.
- Reset cùng một state thủ công ở nhiều custom view.
- Observe bằng Fragment lifecycle khi binding chỉ sống theo view lifecycle.
- Giữ adapter callback truy cập binding sau `onDestroyView()`.

---

## 11. RecyclerView Adapter

Adapter mặc định dùng `ListAdapter` và `DiffUtil`:

```kotlin
class ExampleAdapter(
    private val onItemClick: (ExampleItemUi) -> Unit
) : ListAdapter<ExampleItemUi, ExampleViewHolder>(DiffCallback) {

    override fun onBindViewHolder(holder: ExampleViewHolder, position: Int) {
        holder.bind(getItem(position))
    }

    private object DiffCallback : DiffUtil.ItemCallback<ExampleItemUi>() {
        override fun areItemsTheSame(old: ExampleItemUi, new: ExampleItemUi) =
            old.id == new.id

        override fun areContentsTheSame(old: ExampleItemUi, new: ExampleItemUi) =
            old == new
    }
}
```

Quy tắc:

- Không expose `MutableList` nội bộ.
- Không gọi `notifyDataSetChanged()` nếu diff có thể giải quyết.
- Callback trả item hoặc stable ID, không chỉ trả position.
- Kiểm tra `bindingAdapterPosition != RecyclerView.NO_POSITION` khi đọc position trong click callback.
- Adapter không gọi ViewModel hoặc repository trực tiếp.
- Các adapter giống nhau nên được tổng quát hóa khi abstraction vẫn dễ hiểu.

---

## 12. Naming và code style

- Class/interface/object: `PascalCase`.
- Function/property: `camelCase`.
- Constant: `UPPER_SNAKE_CASE`.
- Android resource: `snake_case`.
- Collection dùng tên số nhiều: `projects`, `selectedTags`, `recipients`.
- Hàm action dùng động từ: `loadProjects`, `selectProject`, `submit`.
- Boolean đọc như câu hỏi: `isLoading`, `canSubmit`, `hasNextPage`.
- UI model có suffix `Ui` khi cần phân biệt với domain/data model.
- DTO chỉ tồn tại ở data boundary.
- Tránh wildcard import.
- Tránh scope function lồng sâu làm receiver không rõ.
- Tránh abbreviation không phổ biến.
- Không dùng text hiển thị làm định danh.
- Không tạo base class/generic abstraction chỉ để giảm vài dòng code.
- Ưu tiên sử dụng apply, run, let, with() theo đúng từng trường hợp để code thành khối,

---

## 13. Dependency injection

Dependency phải được đăng ký ở composition root.

Koin:

```kotlin
single<ExampleRepository> { ExampleRepositoryImpl(get(), get()) }
factory { SubmitExampleUseCase(get()) }
viewModel { ExampleViewModel(get(), get()) }
```

Hilt:

```kotlin
@Binds
abstract fun bindExampleRepository(
    implementation: ExampleRepositoryImpl
): ExampleRepository
```

Quy tắc:

- ViewModel phụ thuộc interface/use case, không tự khởi tạo implementation.
- Không dùng service locator trực tiếp bên trong business class.
- Scope phải phù hợp vòng đời.
- Dependency chỉ đăng ký khi implementation đã tồn tại và feature có thể chạy.

---

## 14. Error handling

Error cần được phân loại tối thiểu:

- Network/unavailable.
- Authentication/authorization.
- Validation/business error.
- Server/API error.
- Unknown error.

Data layer chuyển error kỹ thuật thành error domain. ViewModel quyết định state/effect. View chỉ quyết định cách hiển thị.

Không được:

- Catch rỗng.
- Nuốt exception mà không cập nhật state hoặc log theo convention dự án.
- Hiển thị raw stack trace/backend message nhạy cảm cho người dùng.
- Để loading mãi khi request lỗi.
- Phát cùng một popup lỗi nhiều lần do LiveData/state replay.

---

## 15. Test bắt buộc

### ViewModel unit test

Mỗi feature tối thiểu cần test:

1. Initial state.
2. Load thành công.
3. Load lỗi và loading được tắt.
4. Action cập nhật đúng immutable state.
5. Dữ liệu phụ thuộc được reset đúng khi parent selection thay đổi.
6. Validation chặn submit không hợp lệ.
7. Submit tạo đúng request payload.
8. Submit thành công phát đúng effect.
9. Submit lỗi không làm mất form.
10. Load-more nối dữ liệu đúng và không gọi khi hết trang.
11. Request/query cũ không ghi đè request/query mới.
12. Event một lần không bị xử lý lặp ngoài chủ ý.

Ưu tiên fake repository tự viết:

```kotlin
class FakeExampleRepository : ExampleRepository {
    var submittedCommand: SubmitExampleCommand? = null
    var loadResult: Result<List<ExampleItem>> = Result.success(emptyList())

    override suspend fun submit(command: SubmitExampleCommand) {
        submittedCommand = command
    }
}
```

Fake thường dễ đọc và ổn định hơn mock cho repository contract nhỏ.

### Mapper/repository test

- DTO null/optional được map đúng.
- ID và display text không bị nhầm.
- Error wrapper được map đúng loại.
- Request optional field giữ đúng semantics absent/null/empty.

### UI test khi cần

- State quan trọng render đúng.
- Click phát đúng Action.
- Loading/validation visibility đúng.
- Recreate/rotation không mất state cần giữ và không lặp effect cũ.

Không viết test chỉ để tăng coverage mà không kiểm tra hành vi.

---

## 16. Definition of Done

Feature chỉ được xem là hoàn tất khi:

- [ ] Đúng yêu cầu nghiệp vụ và API contract.
- [ ] Có một nguồn state rõ ràng.
- [ ] State public immutable.
- [ ] View chỉ gửi Action và render State/Effect.
- [ ] ViewModel không phụ thuộc Android View/Fragment/Activity.
- [ ] Repository dependency qua interface hoặc abstraction hiện có phù hợp.
- [ ] Không dùng singleton event bus cho state của feature.
- [ ] Không chọn entity bằng display name khi có ID.
- [ ] Loading/error/empty/content state được xử lý.
- [ ] Cancellation và request race được xử lý phù hợp.
- [ ] Adapter không expose mutable collection.
- [ ] Binding/listener/adapter được giải phóng đúng lifecycle.
- [ ] DI và navigation được nối đầy đủ nếu nằm trong phạm vi.
- [ ] Không còn placeholder hoặc `TODO` làm feature không chạy, trừ blocker được ghi rõ.
- [ ] Unit test cho business flow chính đã có và chạy thành công.
- [ ] Build/lint/test liên quan đã chạy trong khả năng của workspace.
- [ ] Không sửa file ngoài phạm vi nếu không cần.

---

## 17. Cách AI phải bàn giao kết quả

Phần trả lời cuối phải ngắn gọn nhưng có đủ:

1. Kết quả đã triển khai.
2. Các file chính đã tạo/sửa.
3. Kiến trúc và luồng dữ liệu chính.
4. Test/build/lint đã chạy và kết quả.
5. Giả định hoặc phần chưa thể xác minh.
6. Việc còn lại chỉ khi thực sự còn blocker hoặc ngoài phạm vi.

Không chỉ nói “đã hoàn thành”. Phải cho người dùng biết feature được nối ở đâu, test thế nào và có giới hạn gì.

---

## 18. Những anti-pattern bị cấm mặc định

Không sử dụng các pattern sau nếu không có lý do được ghi rõ:

- Fragment/Composable gọi repository trực tiếp.
- ViewModel giữ `Activity`, `Fragment`, binding hoặc View.
- Public `MutableLiveData`/`MutableStateFlow`.
- State chứa `MutableList` hoặc field `var` có thể bị sửa bên ngoài.
- Adapter là nguồn dữ liệu nghiệp vụ chính.
- Global singleton event bus để truyền state trong một feature.
- LiveData state dùng làm event một lần mà không có cơ chế consume đúng.
- Callback từ View truyền vào ViewModel chỉ để nhận kết quả API.
- Chọn entity theo display name thay vì ID.
- Một pagination object dùng chung cho nhiều nguồn dữ liệu.
- Coroutine launch lồng không cần thiết.
- Catch rỗng hoặc nuốt `CancellationException`.
- `notifyDataSetChanged()` cho mọi thay đổi khi có stable list/diff.
- Repository/ViewModel trả trực tiếp mutable collection cho View.
- Tạo use case rỗng chỉ chuyển tiếp repository call.
- Tạo `BaseViewModel` khổng lồ chứa nghiệp vụ của nhiều domain.

---

## 19. Nguyên tắc cuối cùng

Khi phải lựa chọn giữa code “thông minh” và code dễ hiểu, ưu tiên code dễ hiểu.

Một feature tốt phải cho phép lập trình viên mới trả lời nhanh bốn câu hỏi:

1. State của màn hình nằm ở đâu?
2. Người dùng thao tác thì Action đi đâu?
3. API được gọi qua lớp nào?
4. Hành vi quan trọng được test ở đâu? Những hành vi nào quan trọng phải có comment giải thích.
5. Dev khác nhình vào cũng dễ hiểu?
6. Sau này nhìn lại dễ bảo trì và fix bug?

Nếu bốn câu hỏi này không có câu trả lời rõ ràng, kiến trúc cần được đơn giản hóa trước khi mở rộng feature.
