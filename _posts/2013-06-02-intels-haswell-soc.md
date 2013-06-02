---
layout: post
title: "Intel's Haswell SoC"
description: ""
category: computer 
tags: [cpu, instruction, haswell, tsx]
---
{% include JB/setup %}

## Thay đổi trong kiến trúc tập lệnh Haswell
Haswell giới thiệu bốn họ chỉ dẫn lệnh mới. Đầu tiên là AVX2, một mở rộng
256-bit của tập lệnh SIMD dành cho số nguyên trước đây. Về bản chất nó là sự bổ
sung các lệnh số dấu chấm động AVX. AVX2 thêm vào các lệnh hoán vị và dịch
vector, cũng như các lệnh tải dữ liệu từ các vùng địa chỉ
không liên tục nhau (gather). Gather là tính năng cốt yếu để các trình biên dịch có
thể tận dụng các lệnh SIMD rộng hơn (với AVX2 có thể có những thành phần 32
byte).

Về phía dấu chấm động, phần mở rộng Nhân-Cộng tích hợp (Fused Multiply Add -
FMA) [^1] của Intel bao
gồm các lệnh 256-bit và 128-bit. So với các lệnh nhân và cộng được thực hiện
riêng rẽ như ở SSE, FMA theo lý thuyết có thể tăng thông lượng lên gấp đôi.
Thêm vào đó, các lệnh tích hợp loại bỏ bước làm tròn trung gian, giúp tăng độ
chính xác với một số giải thuật xấp xỉ.

Phần mở rộng thứ ba nhỏ hơn và tập trung nhiều hơn vào thao tác bit với số nguyên
(interger bit manipulation - BMI) dành cho mật mã và thao tác với các gói tin.
Intel cũng thêm một lệnh di chuyển theo đầu-to (MOVBE), có thể chuyển đổi qua
lại từ định dạng đầu-nhỏ (little endian) truyền thống của x86. Nó đã được giới
thiệu đầu tiền ở Atom, sau này được thêm vào để đảm bảo tương thích và
cải thiện hiệu năng với các ứng dụng nhúng.

Phần mở rộng kiến trúc tập lệnh đáng chú ý nhất là TSX [^2], đã được giới thiệu
trước đó ở bài [Haswell's transactional
memory](http://arstechnica.com/business/2012/02/transactional-memory-going-mainstream-with-intel-haswell/)
Nói ngắn gọn, TSX tách rời quá trình thực hiện một cách chính xác với các chương
trình đa luồng. Lập trình viên có thể viết các đoạn mã đơn giản dễ dàng gỡ lỗi,
trong khi phần cứng thực hiện chính xác sự đồng thời.

Coarse-grained locking (ví dụ như khóa toàn bộ 1 cấu trúc dữ liệu) 
phát triển đơn giản, đặc biệt là với chương trình đơn luồng. Tuy nhiên
fine-grained locking (ví dụ khóa một phần của cấu trúc dữ liệu như một nút trên
B-tree) thì có hiệu năng cao hơn. Lướt qua khoá phần cứng (Hardware Lock
Elision - HLE) [hle][] sử dụng các lệnh tiền tố gợi ý (hint prefixes) để cung cấp hiệu năng và thông
lượng của fine-grained locking cho chương trình một cách trong suốt, thậm chí khi lập trình viên sử dụng coarse-grained lock.

Giao dịch bộ nhớ giới hạn (Restricted Transactional Memory - RTM) [^3] là một mô
hình lập trình mới trong đó người ta phải chỉ ra các giao dịch một cách tường minh
qua các chỉ dẫn lệnh mới. Những giao dịch này có thể mở rộng các cấu trúc dữ
liệu phức tạp và có thể được tổ hợp lại dễ dàng. Tuy nhiên, việc này yêu cầu
liên kết đến các thư viện mới có sử dụng RTM hoặc có thể phải viết lại phần mềm
để tận dụng được đầy đủ các lợi ích.

Các phần mở rộng mới về kiến trúc tập lệnh rõ ràng là sự thay đổi lớn nhất ở
front-end của Haswell. Ở mức cao hơn, truy xuất và giải mã chỉ dẫn lệnh tương
tự với Sandy Bridge nhưng có một số cải tiến đáng lưu ý.

Bộ tiên đoán rẽ nhánh ở Haswell đã được cải tiến, mặc dù Intel không 
cung cấp thông tin chi tiết. Cache lệnh vẫn là kết hợp 8 đường (8-way
associative), 32KB, được chia sẻ bởi 2 luồng. Tương tự, TLBs vẫn có cùng
dung lượng. Sự thay đổi chính là tăng cường thao tác với cache miss và truy xuất trước để sử
dụng tốt hơn các tài nguyên đã có. Truy xuất lệnh từ cache lệnh vẫn là 16B một
chu kỳ.


<img
src="http://cdn.arstechnica.net/wp-content/uploads/2013/04/HSW_IMG_1.png"
style="float:middle" />


Quá trình giải mã trên Haswell giống với Sandy Bridge. Có bốn bộ giải mã lệnh chính
nhận các lệnh x86 và sinh ra các uops đơn giản. Cái đầu tiên có thể sinh ra 1-4
uops và ba bộ giải mã mỗi cái có thể sinh ra một uops. Giống Sandy Bridge, có
sự loại bỏ các phép toán tích hợp so sánh+nhảy và con trỏ stack. Haswell uops
cache cũng giống như cũ, với 32 tập hợp trên 8 cache line. Mỗi cache line có
thể chứa đến 6 uops.

Hàng đợi uops đã được thiết kế lại để cải thiện hiệu năng đơn luồng. Sandy
Bridge có hai hàng đợi 28 mục, mỗi cái cho 1 luồng. Tuy nhiên, ở Ivy Bridge hai
hàng đợi đã được kết hợp lại thành một cấu trúc với 56 mục. Lợi ích quan trọng
nhất của việc này là khi chỉ có 1 luồng được thực hiện trên Ivy Bridge hay
Haswell, toàn bộ 56 uops đều được sử dụng.


[^1]: FMA là các lệnh tính toán các phép nhân và cộng cùng một lúc, ví dụ như
có thể tính a+bxc với chỉ một lệnh máy. Tập lệnh FMA có trọng Haswell là FMA3.
Thông tin thêm có thể xem ở tài liệu của Intel.
[^2]: TSX - Transactional Synchronization Extensions. Với TSX, các luồng xử lý
không cần phải quan tâm đến việc khóa các dữ liệu được chia sẻ, mà nó chỉ phải
bắt đầu một giao dịch (transaction) mới trước khi bước vào thực hiện bất kỳ
thay đổi nào, và sau khi xong, xác nhận (commit) giao dịch. Ở Haswell, Intel
cung cấp TSX dưới hai dạng: HLE và RTM.
[^3]: Với HLE, ở trước khi yêu cầu khóa hoặc mở khóa, CPU thực hiện một lệnh
đặc biệt gọi là prefixes instruction. Lệnh này bản thân nó không thực hiện gì
nhưng nó làm cho lệnh yêu cầu khóa sau đó trở thành thành công (tức là luồng
thực hiện bị đánh lừa rằng nó có thể yêu cầu khóa mặc dù có thể khóa lúc đó đã
bị luồng khác nắm giữ, cũng vì thế mà nó mới có thể là "lướt qua" khóa phần
cứng). Như vậy, có thể có nhiều luồng cùng thao tác với dữ
liệu bị khóa cùng một lúc, nếu trong quá trình thực hiện có xung đột xảy ra thì
giao dịch sẽ bị hủy bỏ, nếu không thì giao dịch vẫn tiếp tục một cách bình
thường. Nhờ có HLE mà các chương trình cũ vẫn có thể tận dụng TSX mà không cần
phải thay đổi.

Nguồn: [Arstechnica](http://arstechnica.com/gadgets/2013/05/a-look-at-haswell/)

#### Tài liệu tham khảo
1. [Transaction memory going mainstream with Intel
Haswell](http://arstechnica.com/business/2012/02/transactional-memory-going-mainstream-with-intel-haswell/)
2. [Intel® Advanced Vector Extensions Programming
Reference](http://software.intel.com/sites/default/files/m/9/2/3/41604)
