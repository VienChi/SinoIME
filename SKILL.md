---
name: sinoime-input-method-architecture
description: Hiểu và chỉnh sửa frontend bộ gõ SinoIME cho macOS. Dùng skill này khi làm việc với xử lý input, phiên librime, giao diện ứng viên, cấu hình, vòng đời ứng dụng, các lệnh installer, hoặc phối hợp backend/frontend trong repo này.
---

# Skill Kiến trúc bộ gõ SinoIME

Dùng skill này khi thay đổi SinoIME, một frontend InputMethodKit cho librime trên macOS. SinoIME là một bộ gõ (input method), nên tính đúng đắn phụ thuộc vào thứ tự sự kiện, vòng đời phiên (session), hành vi marked text, hình học cửa sổ ứng viên, và bàn giao gọn gàng giữa text client của macOS và librime.

## Sơ đồ repository

Xcode project được tổ chức quanh một app target duy nhất, `SinoIME.app`, cùng các resource đi kèm và plugin librime.

- `SinoIME/Sources/Main.swift`: điểm vào của tiến trình, các lệnh bảo trì dòng lệnh, tạo IMK server, thiết lập app, và khởi động librime toàn cục.
- `SinoIME/Sources/SinoIMEApplicationDelegate.swift`: trạng thái toàn ứng dụng. Sở hữu candidate panel, `SinoIMEConfig` toàn cục, status item, tích hợp cập nhật Sparkle, distributed notification, và setup/finalize librime.
- `SinoIME/Sources/SinoIMEInputController.swift`: controller InputMethodKit chính. Sở hữu một phiên librime đang hoạt động cho mỗi instance controller, nhận sự kiện phím, dịch sự kiện macOS sang sự kiện phím Rime, commit văn bản, cập nhật marked text, và điều khiển candidate panel.
- `SinoIME/Sources/MacOSKeyCodes.swift`: ánh xạ mã phím/cờ modifier của AppKit/Carbon sang ký hiệu và mask phím librime/X11.
- `SinoIME/Sources/SinoIMEConfig.swift`: lớp bọc kiểu (typed wrapper) mỏng trên `RimeConfig`, có fallback base config/schema và cache khi đọc option.
- `SinoIME/Sources/SinoIMETheme.swift`: chuyển cấu hình style của Rime/SinoIME thành font, màu, cờ layout, định dạng ứng viên, và thuộc tính vẽ.
- `SinoIME/Sources/SinoIMEPanel.swift`: panel ứng viên/trạng thái không kích hoạt (nonactivating). Dựng attributed text cho ứng viên, định vị panel gần con trỏ văn bản, xử lý sự kiện chuột phân trang/ứng viên, và ủy quyền hành động chọn về lại input controller.
- `SinoIME/Sources/SinoIMEView.swift`: bề mặt vẽ AppKit tùy chỉnh cho nền preedit/ứng viên, tô sáng, chỉ báo phân trang, văn bản dọc, và hit testing.
- `SinoIME/Sources/ReservedProperty.swift`: giao thức thuộc tính (property) dành riêng của plugin librime cho các gợi ý UI phía frontend như tô sáng chú thích và làm mới UI.
- `SinoIME/Sources/BridgingFunctions.swift`: các hàm hỗ trợ Swift cho struct cầu nối C, chuỗi C tồn tại lâu dài (persistent), gán optional, và tiện ích hình học.
- `SinoIME/Sources/InputSource.swift`: đăng ký Text Input Source, các hàm hỗ trợ enable/disable/select, và tra cứu input source hiện tại.
- `SinoIME/Resources/Info.plist`: metadata đăng ký InputMethodKit, các input mode (`Hans`, `Hant`), tên class controller IMK, tên connection, metadata Sparkle, và thuộc tính input-source.
- `SinoIME/Resources/SinoIME.entitlements`: tắt App Sandbox, bật network client access, và tắt library validation cho dylib/framework đi kèm.
- `SinoIME/SharedSupport`: dữ liệu Rime đi kèm, schema mặc định, dữ liệu OpenCC, và `sinoime.yaml`.
- `SinoIME/librime-*.dylib`, `SinoIME/Frameworks/Linked Frameworks/librime.1.dylib`: thư viện backend và plugin mà frontend sử dụng.

## Khởi động tiến trình

`SinoIMEApp.main()` là điểm vào duy nhất.

1. Đầu tiên kiểm tra tham số dòng lệnh và thoát sớm với các lệnh bảo trì:
   - `--quit`, `--reload`, `--sync`
   - `--install` / `--register-input-source`
   - `--enable-input-source`, `--disable-input-source`, `--select-input-source`
   - `--build`
   - `--ascii`, `--nascii`, `--getascii`
2. Nếu không có lệnh bảo trì nào được xử lý, nó tạo một `IMKServer` dùng `InputMethodConnectionName` từ `Info.plist`.
3. Nó tạo `NSApplication.shared`, gán `SinoIMEApplicationDelegate`, đặt accessory activation policy, và đổi thư mục hiện tại sang `Bundle.main.sharedSupportPath`. Điều này quan trọng vì cấu hình OpenCC/librime có thể dùng đường dẫn từ điển tương đối.
4. Nó chạy một bộ phát hiện khởi động lỗi (problematic-launch detector) nhanh để tránh vòng lặp crash/treo lặp lại do cấu hình sai.
5. Khởi động bình thường gọi:
   - `setupRime()`
   - `startRime(fullCheck: false)`
   - `loadSettings()`
   - `app.run()`
6. Khi `app.run()` trả về, nó gọi `rimeAPI.finalize()`.

## Khởi tạo Librime toàn cục

`SinoIMEApplicationDelegate` sở hữu việc setup librime toàn cục.

- `setupRime()` tạo thư mục dữ liệu người dùng (`~/Library/Rime`) và thư mục log tạm, đặt `RIME_LOG_DIR`, cài đặt notification handler của librime, điền `RimeTraits`, và gọi `rimeAPI.setup(&traits)`.
- Các đường dẫn trait và trường định danh quan trọng:
  - `shared_data_dir`: đường dẫn shared support của app bundle.
  - `user_data_dir`: `~/Library/Rime`.
  - `log_dir`: thư mục tạm `rime.sinoime`.
  - mã/tên/phiên bản phân phối và `app_name = rime.sinoime`.
- `startRime(fullCheck:)` gọi `rimeAPI.initialize(nil)`, rồi `start_maintenance(fullCheck)`. Khi bảo trì thành công, nó triển khai (deploy) `sinoime.yaml` cùng dấu hiệu `config_version`.
- `loadSettings()` mở config gốc `sinoime`, làm mới thiết lập notification/status-icon, và tải theme panel sáng/tối.
- `loadSettings(for schemaID:)` mở config của schema đang hoạt động và, khi có phần `style`, phủ lên style panel riêng của schema. Nếu không, nó dùng lại config gốc.
- `shutdownRime()` đóng config và gọi `rimeAPI.finalize()`.
- `applicationShouldTerminate(_:)` gọi `cleanup_all_sessions()` trước khi kết thúc.

Không khởi tạo/finalize librime từ từng input controller riêng lẻ. Controller sở hữu session; app delegate sở hữu vòng đời của backend.

## Vòng đời Input Controller

`SinoIMEInputController` kế thừa `IMKInputController` và là đối tượng cốt lõi của bộ gõ.

- `init(server:delegate:client:)` lưu client `IMKTextInput` ban đầu, gọi `createSession()`, và đăng ký observer notification cục bộ cho yêu cầu set/report chế độ ASCII.
- `createSession()` chọn bundle identifier của client, tạo một phiên librime bằng `rimeAPI.create_session()`, xóa `schemaId`, và áp dụng các option riêng theo ứng dụng.
- `destroySession()` gọi `rimeAPI.destroy_session(session)` và xóa trạng thái gõ chập (chord typing).
- `deinit` hủy phiên.
- `activateServer(_:)` làm mới client hiện tại, tùy chọn ghi đè bố cục bàn phím từ `keyboard_layout`, xóa cache preedit cục bộ, và cập nhật nhãn trạng thái trên thanh menu từ `ascii_mode` nếu đã có phiên tồn tại.
- `deactivateServer(_:)` ẩn các palette, commit thành phần đang soạn hiện tại vào client, và giải phóng tham chiếu client.
- `commitComposition(_:)` commit dữ liệu librime thô đang chờ qua `client.insertText`, sau đó xóa composition librime.

Controller giữ `client` dưới dạng weak. Luôn guard khi truy cập client. Một bộ gõ có thể được kích hoạt, hủy kích hoạt, hoặc đổi mục tiêu bởi macOS vào những lúc bất tiện.

## Vòng lặp cập nhật Input

Vòng lặp then chốt là `handle(_:client:) -> Bool` trong `SinoIMEInputController`.

1. Đảm bảo có một phiên librime hợp lệ. Nếu `session == 0` hoặc `find_session(session)` thất bại, gọi `createSession()`.
2. Cập nhật client `IMKTextInput` weak từ `sender` khi có thể.
3. Phát hiện thay đổi bundle ID của app client và áp dụng `app_options/<bundle-id>` từ `sinoime.yaml`.
4. Với `.flagsChanged`:
   - Tính các cờ modifier đã thay đổi bằng cách so với `lastModifiers`.
   - Chuyển đổi modifier bằng `SinoIMEKeycode.osxModifiersToRime`.
   - Kiểm tra hoặc suy luận modifier keycode. Việc này bảo vệ trước các công cụ remote desktop gửi keycode giả 0 cho sự kiện modifier.
   - Xử lý caps lock riêng vì librime cần `XK_Caps_Lock` trước khi trạng thái lock-mask thay đổi.
   - Xử lý việc nhả modifier trước khi nhấn để xử lý các sự kiện nhả bị trễ.
   - Cập nhật `lastModifiers` và gọi `rimeUpdate()`.
5. Với `.keyDown`:
   - Bỏ qua phím tắt có Command để ứng dụng client nhận được chúng.
   - Chọn `charactersIgnoringModifiers` hoặc `characters` tùy theo modifier và hành vi ASCII/không-ASCII.
   - Chuyển keycode/ký tự/modifier sang keycode và mask của librime.
   - Gọi `processKey(...)`.
   - Gọi `rimeUpdate()` khi một keycode rime hợp lệ đã được xử lý.
6. Chỉ trả về `true` khi sự kiện đã được xử lý và không nên tiếp tục đến ứng dụng client.

`recognizedEvents(_:)` chỉ trả về mask key-down và flags-changed.

## Chi tiết xử lý phím

`processKey(_:, modifiers:)` là ranh giới hẹp giữa frontend/backend cho phím.

- Trước khi gọi librime, nó đồng bộ các option `_linear` và `_vertical` từ theme panel hiện tại. Hành vi phím mũi tên có thể phụ thuộc vào layout ứng viên và hướng văn bản.
- Nó gọi `rimeAPI.process_key(session, keycode, modifiers)`.
- Nếu librime không xử lý một escape kiểu Vim khỏi command-mode (`Esc`, `Ctrl-C`, `Ctrl-[`) và `vim_mode` được bật, nó ép `ascii_mode` bật lên nếu chưa ở chế độ ASCII.
- Nếu librime xử lý một phím trong khi `_chord_typing` đang hoạt động, các phím in được và modifier sẽ được ghi lại và sau đó được nhả bởi một timer. Các phím không thuộc chord sẽ xóa buffer chord.

`MacOSKeyCodes.swift` được cố tình tập trung hóa. Hãy thêm các chuyển đổi phím vào đó thay vì rải điều kiện keycode khắp controller.

## Rime Update và luồng dữ liệu

`rimeUpdate(clearReservedComments:)` tiêu thụ toàn bộ trạng thái librime hiển thị cho frontend sau khi xử lý phím, phân trang, chọn ứng viên, di chuyển con trỏ, hoặc làm mới UI của plugin.

Trình tự chính:

1. Xóa gợi ý UI của chú thích dành riêng (reserved comment) trừ khi caller yêu cầu giữ lại rõ ràng.
2. `rimeConsumeCommittedText()` gọi `get_commit`, chèn văn bản đã commit vào client, giải phóng struct commit, reset preedit cục bộ, và ẩn panel.
3. `get_status` phát hiện thay đổi schema:
   - tải lại thiết lập riêng theo schema qua app delegate;
   - tính `inlinePreedit` và `inlineCandidate` dùng config panel cộng với option librime (`no_inline`, `inline`);
   - đặt `soft_cursor` của librime là nghịch đảo của inline preedit.
4. `get_context` đọc trạng thái composition và menu:
   - chuỗi preedit;
   - vị trí byte của đoạn được chọn, chuyển sang chỉ số (index) Swift;
   - vị trí con trỏ;
   - văn bản ứng viên, chú thích, nhãn, số trang, cờ trang-cuối, chỉ số được tô sáng.
5. Nó cập nhật marked text qua `show(preedit:selRange:caretPos:)`.
6. Nó cập nhật candidate panel qua `showPanel(...)` trừ khi không có context, trong trường hợp đó nó ẩn các palette.
7. Nó giải phóng context của librime.

Đường đi của văn bản là:

`NSEvent` -> `SinoIMEInputController.handle` -> `processKey` -> `rimeAPI.process_key` -> `rimeUpdate` -> `get_commit`/`get_status`/`get_context` -> `client.insertText` và/hoặc `client.setMarkedText` cùng `SinoIMEPanel.update`.

## Quy tắc Marked Text và Commit

- Văn bản đã commit phải đi qua `client.insertText(_, replacementRange: .empty)`.
- Composition đang hoạt động nên đi qua `client.setMarkedText(_, selectionRange:, replacementRange: .empty)`.
- `show(preedit:selRange:caretPos:)` cache lại preedit, caret, và selected range được đánh dấu (marked) gần nhất để tránh gọi marked-text thừa.
- Khi cấu hình preedit không-inline, controller có thể đặt một khoảng trắng full-width (`U+3000`) làm marked text để các client như iTerm2 không lặp lại (echo) từng ký tự preedit thô.
- `commitComposition(_:)` commit dữ liệu librime thô đang chờ trong lúc deactivation. Điều này quan trọng khi macOS chuyển input source hoặc text client đang focus thay đổi.

Bộ gõ phải thận trọng về thời điểm tiêu thụ sự kiện. Trả về `true` sai sẽ làm mất phím tắt hoặc văn bản của app; trả về `false` sai có thể gây lặp input.

## Luồng Candidate Panel

App delegate tạo một `SinoIMEPanel` dùng chung trong `applicationWillFinishLaunching`. Input controller đang hoạt động tự gán mình vào `panel.inputController` trước khi cập nhật panel.

`showPanel(...)` lấy hình học con trỏ từ `client.attributes(forCharacterIndex:lineHeightRectangle:)`, lưu vào `panel.position`, và gọi `panel.update(...)`.

`SinoIMEPanel.update(...)`:

- lưu trạng thái preedit/ứng viên mới nhất;
- dựng một attributed string duy nhất chứa các dòng preedit và ứng viên;
- áp dụng thuộc tính theme, nhãn ứng viên, chú thích, màu chú thích theo ngữ nghĩa, gợi ý không-ngắt-dòng, và paragraph style;
- cập nhật storage và hướng layout của `NSTextView`;
- ép TextKit layout trước khi đo hình học;
- gọi `SinoIMEView.drawView(...)` cho các đường vẽ nền/tô sáng;
- gọi `show()` để định vị và hiển thị panel.

`SinoIMEPanel.show()`:

- chọn màn hình dựa trên vị trí con trỏ;
- đặt appearance hiệu lực;
- đo văn bản bằng TextKit 2;
- giới hạn panel quá khổ vào phần lớn màn hình và co giãn qua bounds của content view;
- định vị panel bình thường gần con trỏ, có xử lý riêng cho văn bản dọc;
- áp dụng xoay content-view cho chế độ dọc;
- cấu hình nền trong mờ (`NSGlassEffectView` trên macOS 26+, `NSVisualEffectView` với các phiên bản khác);
- đưa panel không-kích-hoạt (nonactivating) lên trên cùng.

Sự kiện chuột và cuộn trên panel được chuyển tiếp lại cho input controller:

- click vào ứng viên -> `selectCandidate(_:)` -> `rimeUpdate()`;
- click/cuộn điều khiển phân trang -> `page(up:)` -> `rimeUpdate()`;
- click vào vị trí preedit -> `moveCaret(forward:)` -> `rimeUpdate()`.

## Vẽ tùy chỉnh

`SinoIMEView` sở hữu mô hình vẽ và hit-testing.

- Nó dùng một `NSTextView` với layout TextKit 2 để đo các đoạn văn bản được render thật.
- `contentRect` và `contentRect(range:)` duyệt qua các đoạn layout văn bản để tính bounds.
- `draw(_:)` dựng các path Core Graphics cho nền panel, nền preedit, nền ứng viên, ứng viên được tô sáng, vùng preedit được tô sáng, viền, bóng đổ, và điều khiển phân trang.
- `shape` cũng được dùng làm mask nền panel và vùng hit-test.
- `click(at:)` ánh xạ điểm chuột về lại offset TextKit và range của ứng viên/preedit.

Khi thay đổi layout panel, giữ đúng thứ tự: đặt attributed text, đặt hướng layout, ép layout, đo, vẽ path, rồi mới show/định vị lại.

## Mô hình cấu hình

`SinoIMEConfig` là một lớp bọc kiểu (typed facade) trên `RimeConfig`.

- `openBaseConfig()` mở config `sinoime`.
- `open(schemaID:baseConfig:)` mở config schema và fallback về config gốc cho giá trị bị thiếu.
- `getBool`, `getDouble`, `getString`, và `getColor` cache lại các lần đọc thành công.
- `getAppOptions(_:)` đọc các option kiểu boolean dưới `app_options/<bundle-id>`.

`SinoIMETheme.load(config:dark:)` đọc `style/*` toàn cục, sau đó các thiết lập preset color scheme tùy chọn. Giá trị theo từng color scheme có thể ghi đè giá trị style cho layout, màu, font, alpha, khoảng cách, và định dạng ứng viên.

Các cờ theme quan trọng:

- `candidate_list_layout`: danh sách ứng viên linear so với stacked.
- `text_orientation`: ngang so với dọc.
- `inline_preedit`, `inline_candidate`: chiến lược marked text so với hiển thị trên panel.
- `translucency`, `mutual_exclusive`, `memorize_size`, `show_paging`.
- `candidate_format`: mẫu (template) dùng `[label]`, `[candidate]`, `[comment]`; `%c` và `%@` kiểu cũ được chuẩn hóa lại.

## Notification và lệnh bên ngoài

Ứng dụng dùng distributed notification cho các lệnh từ tiến trình khác đến instance đang chạy.

- `SinoIMEReloadNotification` -> deploy: tắt Rime, khởi tạo lại, tải lại thiết lập.
- `SinoIMESyncNotification` -> `sync_user_data()`.
- `SinoIMEToggleASCIIModeNotification` -> gửi `SinoIMESetASCIIModeNotification` cục bộ kèm `Bool`.
- `SinoIMEGetASCIIModeNotification` -> gửi yêu cầu report cục bộ; controller đang hoạt động phản hồi bằng `SinoIMEASCIIModeResponse` (`ascii` hoặc `nascii`).
- `kTISNotifySelectedKeyboardInputSourceChanged` -> cập nhật hiển thị status item và finalize các composition bị bỏ lửng.

Cơ chế dự phòng khi finalize rất quan trọng: một số luồng chuyển input-source/macOS có thể không gọi `deactivateServer`. Khi input source được chọn không còn bắt đầu bằng `org.hannom.inputmethod.SinoIME`, app delegate gọi `deactivateServer` trên input controller hiện tại của panel để tránh composition/trạng thái panel bị bỏ lửng.

## Librime Notification Handler

`notificationHandler(...)` được cài đặt bởi `setupRime()` và nhận notification từ backend.

- `deploy/start`, `deploy/success`, `deploy/failure`: hiển thị thông báo cho người dùng.
- `option`: phân tích tên option được bật/tắt, lấy nhãn trạng thái rút gọn và đầy đủ từ librime, cập nhật status icon cho `ascii_mode`, và tùy chọn hiển thị thông báo trạng thái trên panel.
- `property` khi giá trị bắt đầu bằng `_` và chứa `=`: coi đó là thuộc tính (property) dành riêng của frontend và gọi `handleReservedProperty(...)` trên input controller hiện tại của panel trên main actor.
- `schema`: khi notification được bật, trích xuất và hiển thị tên schema.

Các thuộc tính dành riêng (reserved property) hiện gồm:

- `_comment_highlight`: danh sách chỉ số ứng viên phân cách bằng dấu phẩy, vẽ bằng `accent_text_color`.
- `_comment_warning`: danh sách chỉ số ứng viên phân cách bằng dấu phẩy, vẽ bằng `warning_text_color`.
- `_refresh_ui`: yêu cầu `rimeUpdate(clearReservedComments: false)`.

Giá trị reserved-property tương thích kiểu query-string; danh sách phân cách bằng dấu phẩy thuần được phân tích dưới trường `value`.

## Installer và đăng ký Input Source

`SinoIMEInstaller` bọc Text Input Source Services.

- Các input mode là `org.hannom.inputmethod.SinoIME.Hans` và `org.hannom.inputmethod.SinoIME.Hant`; `Hans` là mặc định chính.
- `register()` gọi `TISRegisterInputSource` cho `/Library/Input Library/SinoIME.app` khi chưa có mode SinoIME nào được bật.
- `enable`, `disable`, và `select` thao tác trên các input source của TIS.
- `currentInputSourceID()` đọc `TISCopyCurrentKeyboardInputSource()` và được dùng để điều khiển hiển thị status item và dọn dẹp composition bị bỏ lửng.

`Info.plist` phải nhất quán với `InputSource.swift`: định danh input mode, `InputMethodConnectionName`, và tên class controller là một phần của việc đăng ký bộ gõ với macOS.

## Quy ước Bridge với Backend

Ranh giới Swift/C dùng các kiểu librime được sinh tự động cùng các hàm hỗ trợ trong `BridgingFunctions.swift`.

- Khởi tạo struct librime bằng `.rimeStructInit()` để bộ nhớ được xóa về 0 và `data_size` được đặt đúng.
- Giải phóng struct do librime sở hữu sau khi đọc thành công: commit bằng `free_commit`, status bằng `free_status`, context bằng `free_context`.
- `setCString(_:to:)` nhân bản (duplicate) chuỗi Swift cho trường C. Lưu ý rằng chuỗi C được nhân bản là cấp phát thủ công.
- `RimeStringSlice.asString` phải tôn trọng `.length`; không thay bằng `String(cString:)` cho các nhãn rút gọn.

## Phong cách viết code và comment

Theo phong cách Swift/AppKit hiện có trừ khi có lý do cục bộ chính đáng để làm khác.

- Kiểu (Type) dùng PascalCase: `SinoIMEInputController`, `ReservedPropertyValue`, `SinoIMETheme`.
- Method, property, biến cục bộ, và case của enum dùng camelCase.
- Giữ phong cách viết tắt cục bộ nhất quán với code lân cận: `rimeAPI`, `schemaId`, `currentApp`, `asciiMode`.
- Tên kiểu Boolean nên đọc tự nhiên với `is`, `has`, `can`, `should`, hoặc một danh từ trạng thái rõ ràng khi API hiện có đã dùng kiểu đó.
- Trường bridge C được sinh tự động có thể giữ tên snake_case như `data_size`; dùng SwiftLint suppression hẹp thay vì đổi tên khái niệm API đã sinh ra.
- Ưu tiên `let` cho giá trị không đổi, `private` cho chi tiết cài đặt, và `private(set)` khi kiểu khác cần trạng thái chỉ đọc.
- Giữ vòng đời IMK và xử lý sự kiện trong `SinoIMEInputController`; giữ vòng đời Rime/app toàn cục trong `SinoIMEApplicationDelegate`.
- Giữ truy cập config trong `SinoIMEConfig` và trạng thái hiển thị có thể cấu hình trong `SinoIMETheme`.
- Giữ việc chuyển đổi phím trong `MacOSKeyCodes`; không rải các ánh xạ phím Carbon/Rime thô khắp code xử lý input.
- Giữ trạng thái và định vị candidate panel trong `SinoIMEPanel`; giữ vẽ, hình học, và hit testing trong `SinoIMEView`.

Tái sử dụng các helper hiện có trước khi thêm abstraction mới.

- Dùng `.rimeStructInit()` cho struct librime cần bộ nhớ xóa về 0 và `data_size`.
- Dùng `setCString(_:to:)` khi gán chuỗi Swift vào struct trait/config của Rime.
- Dùng toán tử `?=` cho việc ghi đè config tùy chọn trong code tải theme/config.
- Dùng `NSRange.empty` cho range rỗng sentinel của dự án.
- Dùng `RimeStringSlice.asString` cho các slice của Rime vì nó tôn trọng độ dài của slice.
- Dùng `SinoIMEKeycode` để chuyển đổi phím macOS sang Rime.
- Mở rộng `ReservedPropertyValue` để phân tích reserved-property thay vì thêm parsing chuỗi rời rạc.
- Chỉ thêm helper dùng chung khi nhiều nơi gọi cần cùng một hành vi không tầm thường. Tránh bọc một biểu thức đơn giản, rõ ràng.

Phong cách comment cố tình tối giản.

- Giữ phần header file đơn giản mà dự án đang dùng.
- Giữ nguyên các comment chỉ thị SwiftLint đúng chỗ cần thiết.
- Dùng tiếng Anh cho các comment được giữ lại.
- Comment nên giải thích lý do (why), ràng buộc thứ tự, quyền sở hữu, hoặc điểm đặc thù của platform/backend. Không nhắc lại những gì dòng tiếp theo đã làm.
- Xóa các câu lệnh print debug đã comment và tracing tạm thời thay vì giữ lại trong mã nguồn.
- Giữ comment gần thứ tự sự kiện IMK/librime, ràng buộc đo lường TextKit, chuyển đổi tọa độ ở chế độ dọc, quyền sở hữu bộ nhớ C, và hợp đồng (contract) plugin/frontend.
- Ưu tiên một comment giải thích gọn gàng hơn là liệt kê ví dụ từng nhánh dài dòng, trừ khi ví dụ đó ngăn được một hồi quy (regression) dễ xảy ra.

## Bất biến của bộ gõ (Invariant)

Ghi nhớ các bất biến sau cho mọi thay đổi:

- Vòng đời librime toàn cục thuộc về app delegate; vòng đời session thuộc về input controller.
- Mọi đường xử lý sự kiện phím làm thay đổi trạng thái librime nên gọi `rimeUpdate()` đúng lúc trạng thái frontend cần được tiêu thụ.
- Không tiêu thụ phím tắt Command trong nhập văn bản bình thường; để ứng dụng client xử lý chúng.
- Deactivation phải ẩn panel và commit hoặc xóa composition đang hoạt động để không có marked text hay panel nào bị bỏ lửng.
- Luôn guard trước client `IMKTextInput` nil hoặc cũ (stale).
- Chuyển offset byte của librime sang index chuỗi Swift trước khi dựng giá trị `NSRange`.
- Giữ các lệnh gọi giải phóng `get_context`, `get_status`, và `get_commit` đi cùng với các lần đọc thành công.
- Giữ các option riêng theo ứng dụng khi tạo session và khi bundle của client đang focus thay đổi.
- Hình học candidate panel phụ thuộc vào kết quả layout của TextKit. Tránh đo lường trước khi layout được ép thực thi.
- Văn bản dọc ảnh hưởng đến hành vi phím, hướng layout, xoay nội dung, định vị panel, và hướng phân trang khi cuộn.
- `inlinePreedit` và `inlineCandidate` được xác định chung bởi config theme và option của librime.
- Trạng thái panel dùng chung luôn phải trỏ đến input controller đang hoạt động trước khi cập nhật ứng viên hoặc thao tác chuột.

## Các khu vực thường thay đổi

Với thay đổi xử lý phím:

1. Bắt đầu từ `SinoIMEInputController.handle` và `processKey`.
2. Đặt các ánh xạ phím tái sử dụng được vào `MacOSKeyCodes.swift`.
3. Giữ đúng thứ tự modifier và hành vi caps-lock.
4. Kiểm tra ngữ nghĩa tiêu thụ sự kiện.

Với thay đổi giao diện ứng viên:

1. Bắt đầu từ `SinoIMEPanel.update` để định hình text/attributes/data.
2. Dùng `SinoIMETheme` cho các giá trị style có thể cấu hình.
3. Dùng `SinoIMEView` cho hình học, vẽ, và hit testing.
4. Kiểm thử các trạng thái ngang, linear, dọc, phân trang, inline preedit, và không có ứng viên.

Với thay đổi cấu hình:

1. Chỉ thêm các lần đọc vào `SinoIMETheme` hoặc `SinoIMEConfig` ở nơi giá trị đó thuộc về.
2. Giữ nguyên hành vi config gốc và fallback theo schema.
3. Cân nhắc việc tải theme sáng/tối riêng biệt.

Với thay đổi vòng đời hoặc lệnh:

1. Bắt đầu từ `Main.swift` cho hành vi dòng lệnh.
2. Bắt đầu từ `SinoIMEApplicationDelegate` cho observer toàn app, setup Rime, hành vi status item, và kết thúc ứng dụng.
3. Giữ tên distributed notification ổn định trừ khi mọi nơi gọi đã được cập nhật.

Với phối hợp plugin librime/frontend:

1. Thêm key dành riêng vào `ReservedPropertyKey`.
2. Phân tích giá trị trong `ReservedPropertyValue` hoặc trong `handleReservedProperty`.
3. Áp dụng hiệu ứng UI trong `SinoIMEInputController` hoặc `SinoIMEPanel`, tùy theo trạng thái đó thuộc về session hay rendering.
4. Giữ nguyên hành vi `_refresh_ui` cho việc vẽ lại do plugin điều khiển.

## Danh sách kiểm tra khi xác thực (Validation)

Khi có thể, xác thực bằng chẩn đoán build của Xcode hoặc một lần build Xcode đầy đủ. Với thay đổi hành vi, hãy tự tay kiểm thử:

- kích hoạt/hủy kích hoạt input trong nhiều ứng dụng;
- gõ, commit, hủy, và chuyển input source giữa lúc đang soạn (composition);
- bật/tắt chế độ ASCII và báo cáo trạng thái;
- chuyển schema và tải lại style riêng theo schema;
- chọn ứng viên bằng phím số và bằng chuột;
- phân trang bằng phím, chuột, và cuộn;
- preedit inline và không-inline;
- layout ứng viên dọc và linear;
- lệnh deploy/reload và sync;
- dọn dẹp khi thoát app/đăng xuất.

Lỗi bộ gõ thường xuất hiện dưới dạng văn bản bị lặp, phím tắt bị mất, candidate panel bị bỏ lửng, marked text cũ (stale), hoặc trạng thái riêng của session rò rỉ giữa các app client. Hãy kiểm thử quanh các kiểu lỗi đó trước tiên.
