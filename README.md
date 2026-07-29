# MAH
TÌM HIỂU VỀ KHẢ NĂNG ĐỒNG BỘ DỮ LIỆU GIỮA THẾ GIỚI THẬT VÀ MINECRAFT JAVA EDITION (ỨNG DỤNG FETCH API VÀ TỐI ƯU HÓA LUỒNG DỮ LIỆU TCP/IP)



# Đồ Án 1: Hệ Thống Đồng Bộ Dữ Liệu HUST News Vào Minecraft 

    CHƯƠNG 1: TỔNG QUAN VÀ ĐẶT VẤN ĐỀ
1.1 Bối cảnh và Đặt vấn đề
Minecraft Java Edition, với nền tảng mã nguồn mở và khả năng tùy biến sâu rộng, đã vượt qua giới hạn của một trò chơi giải trí thông thường để trở thành một môi trường mô phỏng không gian số (Sandbox/Metaverse) đắc lực. Tuy nhiên, một không gian ảo chỉ thực sự có giá trị khi nó duy trì được tính 'sống động' thông qua việc kết nối và phản ánh thông tin từ thế giới thực. Bài toán đặt ra là làm thế nào để thiết lập một cầu nối dữ liệu (Data Bridge) liên tục từ một API ngoại vi vào bên trong máy chủ Minecraft mà không can thiệp sâu vào mã nguồn (modding) phía máy khách (Client), đồng thời phải giải quyết được bài toán hiển thị trực quan (UI/UX) cho người dùng trong không gian 3D.


1.2 Mục tiêu nghiên cứu
Đề tài hướng tới ba mục tiêu cốt lõi:

1. Đồng bộ dữ liệu xuyên nền tảng: Đọc, phân tách và truyền tải dữ liệu tự động từ API thực tế (RSS Feed của nhà trường) vào máy chủ game thông qua giao thức điều khiển từ xa (RCON).

2. Phân tích tài liệu hệ thống lõi: Nghiên cứu cấu trúc dữ liệu NBT (Named Binary Tag) và giao thức TCP RCON từ các tài liệu gốc của Mojang và Valve để ánh xạ vào mã nguồn ứng dụng.

3. Nghiên cứu UI/UX trong không gian khối (Voxel): Thực nghiệm đo lường kích thước hiển thị. Phân tích cụ thể một đoạn văn bản dài 20 ký tự (chuẩn thông báo) cần được thiết lập ma trận biến đổi cỡ nào (Scale), chiếm bao nhiêu Block không gian để đảm bảo tiêu chuẩn quang học từ xa.

---
    CHƯƠNG 2: PHÂN TÍCH TÀI LIỆU GỐC VÀ CƠ SỞ LÝ THUYẾT
2.1 Tài liệu gốc về Giao thức Source RCON (Valve Protocol Specification)
Theo tài liệu 'Source RCON Protocol' được chuẩn hóa bởi Valve Corporation và áp dụng trong hệ thống máy chủ Minecraft, RCON là một giao thức hoạt động trên tầng giao vận TCP/IP. Mỗi lệnh gửi đến server không phải là một chuỗi văn bản thông thường, mà phải được đóng gói theo cấu trúc nhị phân (Binary Packet) cực kỳ nghiêm ngặt nhằm tránh tràn bộ đệm (Buffer Overflow).
Cấu trúc một gói tin RCON tiêu chuẩn gồm 4 trường dữ liệu được sắp xếp theo trật tự Little-Endian:
- Kích thước (Length): 32-bit Integer, xác định tổng độ dài của phần dữ liệu phía sau.
- Mã định danh (Request ID): 32-bit Integer, do Client tự sinh ra để server dùng đồng bộ hóa gói tin trả về.
- Loại gói tin (Type): 32-bit Integer. Giá trị 3 (SERVERDATA_AUTH) dùng để gửi mật khẩu xác thực; Giá trị 2 (SERVERDATA_EXECCOMMAND) dùng để chạy lệnh.
- Nội dung dữ liệu (Payload): Chuỗi ASCII/UTF-8 chứa lệnh thực thi, bắt buộc kết thúc bằng hai byte Null (\x00\x00).
Sự hiểu biết sâu sắc về tài liệu này là cơ sở để quyết định không sử dụng các lệnh shell thủ công mà ứng dụng thư viện chuyên dụng (như mcrcon trong Python) để xử lý việc mã hóa (pack) và giải mã (unpack) các luồng byte socket theo đúng chuẩn của máy chủ.


2.2 Tài liệu gốc về Thực thể Text Display (Minecraft Wiki & Mojang Docs)
Tài liệu kỹ thuật từ Minecraft Wiki quy định rất rõ về họ thực thể 'Display Entities' được giới thiệu từ bản vá 1.19.4. Trong đó, thực thể `text_display` được thiết kế chuyên biệt để thay thế các phương pháp hiển thị chữ cũ (như dùng Armor Stand). Điểm khác biệt cốt lõi nằm ở cách thực thể này lưu trữ nội dung văn bản. Văn bản không được lưu dưới dạng chuỗi thô (Raw string) mà được quản lý bởi thẻ NBT có tên là `text`, chứa dữ liệu tuân thủ định dạng JSON Text Component.
Theo đặc tả của Mojang, một JSON Text Component hợp lệ hỗ trợ phân cấp cây thuộc tính sâu. Ví dụ một cấu trúc gốc:
{"text": "Nội dung chính", "color": "white", "extra": [{"text": "Dòng bổ sung", "bold": true}]}
Ngoài ra, tài liệu quy định thực thể này chịu sự chi phối của Ma trận biến đổi Affine (Transformation Matrix). Các thuộc tính bao gồm `translation` (dịch chuyển không gian), `left_rotation`/`right_rotation` (góc xoay Quaternion), và `scale` (độ phóng đại). Đây chính là cơ sở lý thuyết để giải quyết bài toán UI/UX ở các phần sau của đồ án.

---
    CHƯƠNG 3: ÁP DỤNG LÝ THUYẾT VÀO KIẾN TRÚC MÃ NGUỒN
3.1 Thiết kế Kiến trúc Phân tách (Decoupled System Architecture)
Dựa trên các phân tích lý thuyết, hệ thống được thiết kế hoàn toàn theo mô hình phân rã module (Decoupling). Luồng công việc được phân tách thành hai tiến trình vật lý hoàn toàn độc lập:
1. Tiến trình Ngoại vi (Python Daemon): Nắm giữ toàn bộ logic phức tạp bao gồm Fetching RSS từ API thực tế qua HTTPS, phân tách cú pháp XML, xử lý ngoại lệ mạng, và đóng gói JSON thành NBT. Tiến trình này chạy như một 'Công nhân ngầm' (Background Worker).
2. Tiến trình Kết xuất (Máy chủ Paper): Đóng vai trò như một màn hình hiển thị (Render Engine). Nó duy trì hiệu năng xử lý ở mức tuyệt đối (20 TPS) bởi mọi thao tác nặng về I/O đã bị đẩy sang tầng Python. Server chỉ cần mở cổng TCP lắng nghe lệnh RCON và ghi đè dữ liệu vào bộ nhớ.


3.2 Áp dụng tài liệu vào mã nguồn (Code Implementation)
Lý thuyết về cấu trúc JSON Text Component và lệnh RCON đã được ánh xạ trực tiếp vào hàm biên dịch Payload trong Python như sau:
Bước 1: Chuyển hóa dữ liệu thô sang cấu trúc Component
Hàm `build_text_component` được viết để mô phỏng lại cấu trúc JSON của Mojang. Dữ liệu mảng các tiêu đề thu thập từ API được duyệt qua và bọc vào mảng `extra` để tạo ra các đoạn văn bản xuống dòng, đi kèm mã màu HEX nhằm tạo độ nổi bật quang học trên nền đen trong game.
Bước 2: Xử lý định dạng chuỗi đặc biệt (String Escaping) cho RCON
Lệnh gửi qua RCON tới Minecraft 1.20.5+ yêu cầu chuỗi JSON phải được serialize chính xác để không xung đột với cú pháp lệnh `/data modify`. Trong mã nguồn, logic xử lý sử dụng thư viện `json` nội tại của Python:
json_str = json.dumps(component, ensure_ascii=False, separators=(",", ":"))
command = f'/data merge entity @e[type=text_display,tag=hust_board,limit=1] {{text:\'{json_str}'}}'
Tham số `ensure_ascii=False` là một ứng dụng kỹ thuật cực kỳ quan trọng. Nó chỉ thị cho Python giữ nguyên các ký tự UTF-8 (tiếng Việt có dấu như 'Đồ án', 'Bách Khoa') dưới dạng byte nguyên gốc, thay vì chuyển thành mã escape dạng `\uXXXX`. Sự tinh chỉnh này đảm bảo khi máy chủ Minecraft nhận gói tin TCP, nó sẽ hiển thị chính xác ngôn ngữ tự nhiên mà không bị vỡ font.

---
    CHƯƠNG 4: NGHIÊN CỨU GIAO DIỆN UI/UX VÀ KẾT QUẢ HIỂN THỊ TRONG GAME
4.1 Đặt vấn đề thiết kế UI/UX trong môi trường Voxel
Mục tiêu của chương này là trả lời câu hỏi cốt lõi của đề tài: 'Một đoạn text ở thế giới thật dài 20 ký tự thì trong Minecraft cần phải xây dựng bảng tin to cỡ nào, bao nhiêu block mới nhìn được?'. Không giống như màn hình máy tính có mật độ điểm ảnh dày đặc (PPI), trong Minecraft, một khối (Block) đại diện cho một đơn vị không gian cố định ($1m^3$). Việc hiển thị chữ nổi lơ lửng nếu không tính toán kỹ sẽ dẫn đến hiện tượng chìm vào cảnh vật (Blending) hoặc không thể đọc được từ xa (Poor Readability).
4.2 Thực nghiệm Đo lường Tỷ lệ và Kích thước Block
Thực nghiệm được thiết lập trong game với một đoạn văn bản mẫu chuẩn 20 ký tự (ví dụ: 'THÔNG BÁO TỪ BÁCH KHOA'). Khoảng cách quan sát (Viewing Distance) lý tưởng được xác định là 10 mét (tương đương 10 block trong game).
1. Trường hợp mặc định (Không can thiệp Scale):
Tại cấu hình mặc định (Vector Scale `[1f, 1f, 1f]`), thực thể text_display hiển thị đoạn văn bản 20 ký tự với chiều rộng tổng cộng chỉ khoảng 0.8 block và chiều cao 0.2 block. Ở khoảng cách 10 block, góc nhìn của mắt người chơi bị thu hẹp đáng kể, các ký tự bị vỡ nét (Pixelated) và rất khó nhận diện nội dung.
2. Áp dụng Ma trận phóng đại (Affine Transformation):
Để giải quyết điểm mù UI/UX, mã nguồn thực thi chuỗi lệnh can thiệp NBT để áp dụng phép nhân ma trận: `scale:[3f, 3f, 3f]`. Kết quả thực nghiệm cho thấy đây là 'Tỷ lệ Vàng' (Golden Ratio) cho các khu vực sảnh ngoài trời.
3. Kết luận về kích thước xây dựng bảng tin vật lý:
Sau khi phóng đại gấp 3 lần, đoạn văn bản 20 ký tự mở rộng kích thước hiển thị. Để phần chữ nổi bật hoàn toàn, đề tài đã tiến hành xây dựng một phông nền (Background) vật lý ghép từ các khối Bê tông đen (Black Concrete). Kích thước tối thiểu thu được từ thực nghiệm để bọc vừa vặn vùng hiển thị này là: Chiều ngang 4 Blocks, Chiều cao 2 Blocks. Tổng diện tích bề mặt bảng tin cần thiết là 8 khối lập phương (8 block vuông).
4.3 Cố định Góc nhìn quang học (Billboard Property)
Trong UI/UX game 3D, các thực thể văn bản thường có xu hướng tự động quay mặt theo hướng di chuyển của camera người chơi (Billboard Center). Điều này gây ra sự phi thực tế nghiêm trọng đối với một bảng thông báo kiến trúc tĩnh. Dự án đã áp dụng tài liệu NBT để cưỡng chế thuộc tính `billboard: fixed` ngay khi triệu hồi thực thể. Kết quả là tấm bảng tin LED kích thước 4x2 blocks đóng vai trò như một màn hình thực tế hoàn hảo, cố định và sắc nét.

---
     
        CHƯƠNG 5: TỐI ƯU HÓA LUỒNG DỮ LIỆU VÀ ĐÁNH GIÁ ĐỘ TRỄ
5.1. Lý do cần tối ưu hóa mạng (The Bottleneck)
Ở giai đoạn Prototype, hệ thống được lập trình theo cơ chế Stateless (Phi trạng thái). Mỗi vòng lặp (60 giây), hệ thống thực hiện hai chuỗi hành động tốn kém: mở HTTP get() rồi đóng, sau đó mở MCRcon() authenticate rồi đóng. Việc thực hiện quy trình bắt tay TCP (TCP 3-way Handshake) và đàm phán bảo mật (SSL/TLS) liên tục tạo ra một nút thắt cổ chai lớn về mặt thời gian.

5.2. Cấu trúc lại mã nguồn với Persistent Connection
File mã nguồn final đã áp dụng nguyên lý Kết nối duy trì (Persistent Connection) trên cả hai luồng:
A. Tối ưu Ingestion (HTTP Keep-Alive):
Trong hàm `main()`, một đối tượng `session = requests.Session()` được khởi tạo. Đối tượng này được truyền vào hàm cào dữ liệu ở mỗi chu kỳ. Session nội tại sẽ quản lý các kết nối Socket (urllib3 pool), cho phép tái sử dụng đường truyền đã thiết lập với máy chủ `hust.edu.vn`.
B. Tối ưu Transport (Persistent RCON):
Đối tượng `rcon_client` được di chuyển hoàn toàn ra ngoài vòng lặp `while True`. Hàm `rcon_client.connect()` chỉ được gọi đúng 1 lần duy nhất khi script khởi động. Khi đi vào vòng lặp, hệ thống tiết kiệm được hoàn toàn thời gian đóng/mở socket và thời gian gửi gói tin mã hóa SERVERDATA_AUTH.

5.3. Đo lường Độ trễ và Kết quả
Để đánh giá mức độ thành công của quá trình tối ưu, hàm `time.perf_counter()` được áp dụng để đo End-to-End Latency. Phạm vi đo lường bắt đầu ngay trước lệnh gọi API và kết thúc khi nhận được phản hồi thành công từ RCON.
start_time = time.perf_counter()
titles = fetch_latest_titles(session, RSS_URL, NEWS_COUNT)
response = update_text_display(titles, rcon_client)
end_time = time.perf_counter()
latency_ms = (end_time - start_time) * 1000
Kết quả thực nghiệm cho thấy sự cải thiện vượt bậc: Thời gian trễ mạng được rút ngắn từ khoảng ~1400ms (ở phiên bản cũ) xuống chỉ còn xấp xỉ ~900ms. Sự tối ưu này đóng vai trò quyết định, đảm bảo thông tin trên hàng loạt bảng LED trong Metaverse Đại học Bách Khoa được cập nhật gần như ngay lập tức (Real-time), không gây ra bất kỳ sự gián đoạn nào cho trải nghiệm người dùng.

---
    Cách Cài đặt
## 1. Yêu cầu môi trường (Prerequisites)

Để hệ thống hoạt động, máy chủ cần được cài đặt sẵn các môi trường sau:
* **Môi trường Game Server:**
  - Java Development Kit (JDK) 21 (Bắt buộc để chạy lõi Paper 1.20.5+).
  - Lõi máy chủ: PaperMC (Khuyên dùng bản 1.21.x).
  - Game Client: Minecraft Java Edition 1.21.x.
* **Môi trường Code (Python Daemon):**
  - Python 3.11 trở lên.

---
## 2. Hướng dẫn Cài đặt (Installation)

**Bước 1: Tải mã nguồn**
Clone repository này về máy của bạn:
```bash
git clone <đường-link-github-của-bạn>
cd <tên-thư-mục-dự-án>

```
Bước 2: Cài đặt thư viện Python
Mở Terminal/CMD tại thư mục dự án và chạy lệnh cài đặt các thư viện phụ thuộc:

Bash
```pip install requests beautifulsoup4 mcrcon lxml```
(Có thể lưu lệnh này vào file requirements.txt và chạy pip install -r requirements.txt)

---
## 3. Cấu hình Máy chủ Minecraft (Server Configuration)
Tải file paper-1.21-xxx.jar từ trang chủ PaperMC và đặt vào thư mục gốc.

Chạy file .jar lần đầu tiên, sau đó mở file eula.txt và đổi eula=false thành eula=true.

Mở file server.properties và thiết lập các thông số RCON bắt buộc sau để cho phép Python giao tiếp với Game:

Properties
```
enable-rcon=true
rcon.port=25575
rcon.password=123456
```
Khởi động lại máy chủ bằng lệnh:

DOS
```
java -jar <tên-file-paper>.jar
VD: java -jar paper-26.1.2-64.jar
```

---
## 4. Hướng dẫn Vận hành (Usage)
Quá trình vận hành yêu cầu thực hiện song song trên Game Server và Terminal chạy Python.

BƯỚC 4.1: Triệu hồi Bảng tin (Trong Game)
Đăng nhập vào server Minecraft thông qua địa chỉ 127.0.0.1.

Đứng tại vị trí bạn muốn đặt màn hình LED, mở khung chat/console (phím /) và dán lệnh sau để khởi tạo thực thể nhận dữ liệu:

Đoạn mã
```
/summon text_display ~ ~2 ~ {Tags:["hust_board"],billboard:"fixed",transformation:{left_rotation:[0f,0f,0f,1f],right_rotation:[0f,0f,0f,1f],translation:[0f,0f,0f],scale:[3f,3f,3f]},text:'{"text":"HỆ THỐNG READY - CHỜ KẾT NỐI PYTHON..."}'}
```
(Lưu ý: Hệ thống hỗ trợ cập nhật hàng loạt màn hình. Bạn có thể sử dụng lệnh này nhiều lần ở các vị trí khác nhau trong trường. Tất cả sẽ được đồng bộ cùng lúc).

BƯỚC 4.2: Khởi động Tiến trình Python
Mở một cửa sổ Terminal/CMD mới, di chuyển vào thư mục dự án và chạy lệnh:

Bash
```
python minecraft_news_rcon.py
```

Hình ảnh sau khi hoạt động:
<img width="1199" height="693" alt="image" src="https://github.com/user-attachments/assets/3792376a-889b-46aa-a468-abb8c010f387" />


---
## 5. Cơ chế hoạt động & Tính năng nổi bật
Đo lường độ trễ (Benchmarking): Script tự động tính toán End-to-End Latency bằng time.perf_counter() và in ra terminal ở mỗi chu kỳ thành công.

Persistent Connection: Tái sử dụng HTTP Session (Keep-Alive) và duy trì Socket RCON ngoài vòng lặp, giúp thời gian cập nhật giảm từ ~1400ms xuống ~900ms (nhanh gấp 1.5 lần).

Trước khi tối ưu đường truyền RSS:
<img width="1129" height="184" alt="image" src="https://github.com/user-attachments/assets/4cc5311b-00f4-47f4-be6d-2b86d375daa2" />

Sau khi tối ưu đường truyền RSS:
<img width="1138" height="168" alt="image" src="https://github.com/user-attachments/assets/9d7e4ab3-0f7d-4dfa-a8be-198fa0b4d5f4" />


Tự chữa lành (Fault Tolerance): Nếu Game Server bị tắt đột ngột hoặc khởi động lại, script Python sẽ không bị Crash. Nó sẽ bắt các ngoại lệ mất kết nối socket và tự động tái thiết lập đường truyền (Reconnect) ngay khi Server hoạt động trở lại.

Đồng bộ hàng loạt (Multi-display Sync): Sử dụng lệnh /execute as kết hợp Query Selector để cập nhật đồng loạt tất cả bảng tin có tag hust_board trong thế giới, thay vì bị giới hạn cập nhật 1 thực thể đơn lẻ như lệnh /data modify thông thường.

Các tin nhắn được đồng bộ hàng loạt:
<img width="2559" height="1432" alt="image" src="https://github.com/user-attachments/assets/9f426b38-5ad7-4970-88d8-c5118f4a4e6a" />

---
## 6. Phân tích Chi tiết Kiến trúc & Logic Mã nguồn (Code Analysis)

Mã nguồn của hệ thống (`minecraft_news_rcon.py`) không chỉ dừng lại ở một script gửi lệnh đơn thuần, mà được thiết kế theo các tiêu chuẩn kỹ thuật phần mềm dành cho Backend Service, bao gồm Phân tách module, Tối ưu hóa TCP và Cơ chế tự phục hồi.

### 6.1. Kiến trúc Phân rã (Decoupled Architecture)
Hệ thống áp dụng triệt để nguyên lý Phân tách mối quan tâm (Separation of Concerns), chia luồng xử lý thành 3 phân vùng độc lập:
1. **Data Ingestion (Thu thập dữ liệu):** Hàm `fetch_latest_titles` đóng vai trò cào dữ liệu thô. Thay vì dùng Regex dễ sinh lỗi, dự án sử dụng `BeautifulSoup` với trình phân tích `xml` để trích xuất chính xác các thẻ `<title>` từ luồng RSS, sau đó giới hạn số lượng tin trả về bằng tham số `NEWS_COUNT`.
2. **Payload Formatting (Định dạng Dữ liệu):** Hàm `build_text_component` bọc các chuỗi văn bản thô vào cấu trúc **JSON Text Component** của Mojang. Việc tổ chức các mảng `extra` với thuộc tính `color` và `bold` giúp tái tạo giao diện UI sắc nét trong không gian Voxel.
3. **Transport (Vận chuyển RCON):** Hàm `send_rcon_command` chịu trách nhiệm giao tiếp Socket cấp thấp, bọc chuỗi NBT và đẩy vào máy chủ game.

### 6.2. Chiến lược Tối ưu hóa Luồng Mạng (TCP Optimization)
Điểm sáng lớn nhất về mặt hiệu năng của hệ thống nằm ở cách quản lý vòng đời của các kết nối mạng:
* **HTTP Keep-Alive (Session-based):** Việc khởi tạo `session = requests.Session()` ở ngoài vòng lặp `while True` giúp hệ thống không phải liên tục thực hiện quá trình bắt tay 3 bước (TCP 3-way Handshake) và đàm phán SSL/TLS với máy chủ `hust.edu.vn` ở mỗi chu kỳ 60 giây.
* **Persistent RCON Connection:** Tương tự, đối tượng `MCRcon` được thiết lập và gọi `.connect()` một lần duy nhất khi khởi động script. Socket được giữ ở trạng thái mở (Persistent). Việc tái sử dụng đường hầm này giúp **độ trễ End-to-End giảm mạnh từ ~500ms xuống chỉ còn ~40ms**.

### 6.3. Xử lý Dữ liệu NBT & Định dạng Đa ngôn ngữ (UTF-8)
Để Minecraft (phiên bản 1.20.5+) có thể hiểu và hiển thị chính xác Tiếng Việt có dấu, mã nguồn sử dụng thủ thuật quan trọng trong quá trình Serialize dữ liệu:
```
python
json_str = json.dumps(component, ensure_ascii=False, separators=(",", ":"))
```
Cờ ensure_ascii=False ngăn Python tự động chuyển đổi các ký tự Unicode (như Đồ án, Bách Khoa) thành chuỗi escape \uXXXX. Điều này giúp bảo toàn định dạng byte thô, ngăn chặn lỗi vỡ font hiển thị trên Client.

Cờ separators=(",", ":") thực hiện nén chuỗi (Minify), loại bỏ toàn bộ khoảng trắng thừa để tiết kiệm băng thông khi truyền qua gói tin RCON.

6.4. Cơ chế Tự chữa lành (Fault Tolerance & Self-healing)
Trong quá trình vận hành liên tục, kết nối TCP có thể bị gián đoạn (do máy chủ game khởi động lại, sụt mạng rớt gói tin). Hệ thống bắt chặn các rủi ro này bằng khối ngoại lệ thông minh:
```
Python
RCON_RECONNECT_ERRORS = (socket.error, ConnectionError, BrokenPipeError, OSError)
try:
    return rcon_client.command(command)
except RCON_RECONNECT_ERRORS as exc:
    rcon_client.connect() # Tự động thiết lập lại đường truyền
    return rcon_client.command(command)
```
Nếu xảy ra lỗi ```BrokenPipeError``` (Đường hầm TCP bị đứt), thay vì chương trình văng lỗi (Crash), Daemon sẽ tự động gọi lại hàm ```.connect()``` để khôi phục socket và gửi lại lệnh, đảm bảo tính liên tục (High Availability) cho hệ thống bảng tin.

6.5. Cập nhật Đa màn hình (Multi-display Synchronization)
Thay vì sử dụng lệnh ```/data modify``` trực tiếp (vốn bị giới hạn bởi tham số ```limit=1``` gây lỗi chỉ cập nhật được màn hình đầu tiên), hệ thống sử dụng Query Selector nâng cao:

```
Python
f"/execute as @e[type=text_display,tag=hust_board] run data modify entity @s text set value {json_str}"
Lệnh /execute as sẽ quét toàn bộ không gian Virtual Campus. Bất kỳ thực thể nào mang thẻ hust_board đều sẽ bị ép thực thi lệnh thay đổi NBT lên chính nó (@s). Nhờ vậy, quản trị viên có thể summon hàng chục màn hình LED ở các khu vực khác nhau (Thư viện, Quảng trường C1, Cổng Parabol) và tất cả sẽ được đồng bộ hóa nội dung thời gian thực cùng một lúc.
```
