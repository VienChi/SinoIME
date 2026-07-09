# Cách dùng Rime với SinoIME

> Hướng dẫn build SinoIME - bộ gõ frontend của Rime cho macOS

## Tự tay build và cài đặt SinoIME

### Điều kiện tiên quyết

Cài **Xcode 14.0** trở lên từ App Store, để build SinoIME dưới dạng ứng dụng
Universal.

Cài **cmake**.

Tải về từ https://cmake.org/download/

hoặc cài từ [Homebrew](http://brew.sh/):

``` sh
brew install cmake
```

hoặc cài từ [MacPorts](https://www.macports.org/):

``` sh
port install cmake
```

### Lấy mã nguồn

``` sh
git clone --recursive https://github.com/hannom/sinoime.git

cd sinoime
```

Tùy chọn: lấy thêm các plugin của Rime (danh sách repo GitHub):

``` sh
bash librime/install-plugins.sh rime/librime-sample # ...
```

Các plugin phổ biến gồm [librime-lua](https://github.com/hchunhui/librime-lua), [librime-octagram](https://github.com/lotem/librime-octagram) và [librime-predict](https://github.com/rime/librime-predict)

### Cách nhanh: lấy bản librime mới nhất

Bạn có thể bỏ qua hai bước tiếp theo - build Boost và librime - bằng cách
tải bản nhị phân librime mới nhất từ GitHub releases.

``` sh
bash ./action-install.sh
```

Sau khi xong, bạn có thể chuyển sang mục [Build SinoIME](#build-sinoime).

### Cài thư viện Boost C++

Chọn một trong các cách sau.

**Cách 1:** Tải và cài từ mã nguồn.

``` sh
export BUILD_UNIVERSAL=1

bash librime/install-boost.sh

export BOOST_ROOT="$(pwd)/librime/deps/boost-1.84.0"
```

Đặt `BUILD_UNIVERSAL` để báo cho `make` biết ta đang build Boost dưới dạng
nhị phân macOS universal. Bỏ qua việc này nếu chỉ build cho kiến trúc hiện tại.

Sau khi mã nguồn Boost được tải về và một vài thư viện đã build xong,
nhớ đặt biến shell `BOOST_ROOT` trỏ đến thư mục gốc của nó như trên.

Bạn cũng có thể đặt `BOOST_ROOT` trỏ đến một cây mã nguồn Boost có sẵn trước bước này.

**Cách 2:** Cài phiên bản hiện có từ Homebrew:

``` sh
brew install boost
```

**Lưu ý:** với cách này, `SinoIME.app` build ra sẽ không portable (không mang đi máy khác được) vì
nó liên kết đến các thư viện cài cục bộ từ Homebrew.

Tìm hiểu thêm về ảnh hưởng của việc này tại
https://github.com/rime/librime/blob/master/README-mac.md#install-boost-c-libraries

**Cách 3:** Cài từ [MacPorts](https://www.macports.org/):

``` sh
port install boost -no_static
```

### Build SinoIME

* Đảm bảo bạn đã cập nhật đầy đủ các dependency. Nếu bạn clone sinoime bằng lệnh trong hướng dẫn này thì đã xong bước này rồi. Nếu chưa, lệnh sau sẽ cập nhật các submodule.

```
git submodule update --init --recursive
```

* Có một vài biến môi trường bạn có thể khai báo. Danh sách và các giá trị khả dĩ:

``` sh
export BOOST_ROOT="path_to_boost" # bắt buộc
export DEV_ID="Your Apple ID name" # thêm vào nếu muốn codesign, tùy chọn
export BUILD_UNIVERSAL=1 # đặt để build nhị phân universal
export PLUM_TAG=":preset” # hoặc ":extra", tùy chọn, build kèm một bộ công thức plum
export ARCHS='arm64 x86_64' # tùy chọn, nếu không khai báo thì chỉ build cho kiến trúc đang chạy
export MACOSX_DEPLOYMENT_TARGET='13.0' # tùy chọn, phiên bản thấp hơn 13.0 chưa được kiểm thử và có thể không hoạt động đúng
```

* Khi mọi dependency đã sẵn sàng, build `SinoIME.app`:

``` sh
make
```

* Bạn có thể khai báo các biến môi trường trong shell/terminal, hoặc thêm chúng như tham số cho lệnh make. Ví dụ:

``` sh
# cho ứng dụng macOS Universal
make ARCHS='arm64 x86_64' BUILD_UNIVERSAL=1
```

## Cài đặt lên máy Mac của bạn

### Tạo gói cài đặt (Package)

Chỉ cần thêm `package` sau `make`

```
make package ARCHS='arm64'
```

Khai báo `DEV_ID` để tự động xử lý ký mã (code signing) và [công chứng](https://developer.apple.com/documentation/security/notarizing_macos_software_before_distribution) (cần có Apple Developer ID)

Để việc này hoạt động, bạn cần một `Developer ID Installer: (tên/tổ chức của bạn)` và đặt tên/tổ chức đó làm biến môi trường `DEV_ID`.

Để công chứng (notarization) hoạt động, bạn cũng cần lưu thông tin xác thực dưới cùng tên như trên.

```
xcrun notarytool store-credentials 'your name/org'
```

Bạn **không** cần khai báo `DEV_ID` nếu không có ý định phân phối gói cài đặt.

### Cài đặt trực tiếp

**Bạn có thể cần dùng sudo, và nếu không đăng xuất, ứng dụng có thể không hoạt động đúng. Không khuyến khích cài đặt trực tiếp.**

Sau khi build xong, bạn có thể cài và dùng thử ngay trên máy Mac của mình:

``` sh
# SinoIME dưới dạng ứng dụng Universal
make install
```

## Dọn dẹp file build

Sau khi cài đặt hoặc sau một lần thử thất bại, bạn có thể muốn làm lại từ đầu. Trước khi làm vậy, **hãy chắc chắn đã dọn dẹp các file từ lần build trước.**

Để dọn dẹp các file build của **SinoIME**, không đụng đến dependency, chạy:

``` sh
make clean
```

Để dọn dẹp **dependency**, gồm librime, các plugin librime, plum và sparkle, chạy:

``` sh
make clean-deps
```

Để dọn dẹp các **gói cài đặt (package)**, chạy:

``` sh
make clean-package
```

Nếu muốn dọn dẹp tất cả các mục trên, chạy hết.

Vậy là xong, một hành trình bằng lời. Cảm ơn bạn đã gõ chữ cùng SinoIME.
