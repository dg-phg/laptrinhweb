# MÔN: PTƯDTNWEB_DTAP_BT2
## Thông tin sinh viên:
+ **Họ và tên:** Dương Thị Anh Phương
+ **Mã sinh viên:** K235480106056
+ **Lớp:** K235480106056
+ **Trường:** Đại học Kỹ thuật Công nghiệp Thái Nguyên
---
## BÀI TẬP 2













#### 1. sử dụng nodered: dùng node http_in + http_response => tạo api đơn giản

Chương trình của function:
```
// Khai báo Header trả về định dạng JSON
msg.headers = {
    'Content-Type': 'application/json'
};

// Thuật toán / Logic: Tạo dữ liệu bảng điểm sinh viên
const bangDiem = [
    { ma_mon: "INT1001", ten_mon: "Mạng máy tính", tin_chi: 3, diem_so: 8.5 },
    { ma_mon: "INT1002", ten_mon: "Lập trình Web", tin_chi: 3, diem_so: 9.0 },
    { ma_mon: "INT1003", ten_mon: "Cơ sở dữ liệu", tin_chi: 4, diem_so: 8.0 }
];

// Đóng gói JSON trả về client
msg.payload = {
    ok: 1,
    msg: "Lấy bảng điểm thành công",
    student_info: {
        mssv: "K59KMT001",
        ho_ten: "Nguyễn Văn Phương",
        lop: "K59 KMT"
    },
    ds_diem: bangDiem
};

return msg;
```
<img width="892" height="853" alt="image" src="https://github.com/user-attachments/assets/45fb05df-54a5-47ef-b96f-72fadeb8cce0" />
#### 2. cấu hình nginx để web dùng js gọi đc API trên nodered, thuật toán cho api

#### 3. code js vào trang html để gọi đc api trên

```
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hệ Thống Tra Cứu Bảng Điểm</title>
    <!-- Thêm Bootstrap 5 để giao diện hiện đại -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">

    <div class="container py-5">
        <div class="row justify-content-center">
            <div class="col-md-8">
                <div class="card shadow-sm border-0">
                    <div class="card-header bg-primary text-white py-3">
                        <h4 class="mb-0 fw-bold">🎓 Tra Cứu Bảng Điểm Sinh Viên</h4>
                    </div>
                    <div class="card-body p-4">
                        <button class="btn btn-primary btn-lg mb-3" onclick="xemBangDiem()">
                            🔍 Xem Bảng Điểm
                        </button>
                        
                        <p id="status-msg" class="text-muted fst-italic"></p>

                        <!-- Thẻ thông tin sinh viên -->
                        <div id="info" class="alert alert-info border-0 shadow-sm" style="display: none;">
                            <div class="row">
                                <div class="col-md-6"><b>MSSV:</b> <span id="mssv"></span></div>
                                <div class="col-md-6"><b>Họ và tên:</b> <span id="ho-ten"></span></div>
                                <div class="col-md-12 mt-2"><b>Lớp:</b> <span id="lop"></span></div>
                            </div>
                        </div>

                        <!-- Bảng điểm -->
                        <div class="table-responsive">
                            <table id="table-diem" class="table table-hover table-striped align-middle" style="display: none;">
                                <thead class="table-dark">
                                    <tr>
                                        <th>Mã môn</th>
                                        <th>Tên môn học</th>
                                        <th class="text-center">Tín chỉ</th>
                                        <th class="text-center">Điểm số</th>
                                    </tr>
                                </thead>
                                <tbody id="body-diem"></tbody>
                            </table>
                        </div>

                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        async function xemBangDiem() {
            const statusMsg = document.getElementById('status-msg');
            const infoBox = document.getElementById('info');
            const tableDiem = document.getElementById('table-diem');
            const bodyDiem = document.getElementById('body-diem');
            
            statusMsg.innerText = "⏳ Đang tải dữ liệu từ API Node-RED...";

            try {
                const response = await fetch('/api/bang-diem');
                const result = await response.json();

                if (result.ok === 1) {
                    document.getElementById('mssv').innerText = result.student_info.mssv;
                    document.getElementById('ho-ten').innerText = result.student_info.ho_ten;
                    document.getElementById('lop').innerText = result.student_info.lop;

                    bodyDiem.innerHTML = "";
                    result.ds_diem.forEach(item => {
                        const row = `<tr>
                            <td><span class="badge bg-secondary">${item.ma_mon}</span></td>
                            <td class="fw-bold">${item.ten_mon}</td>
                            <td class="text-center">${item.tin_chi}</td>
                            <td class="text-center"><span class="badge bg-success fs-6">${item.diem_so}</span></td>
                        </tr>`;
                        bodyDiem.innerHTML += row;
                    });

                    infoBox.style.display = "block";
                    tableDiem.style.display = "table";
                    statusMsg.innerText = "";
                } else {
                    statusMsg.innerText = "❌ Lỗi: Không lấy được dữ liệu!";
                }
            } catch (error) {
                console.error("Lỗi gọi API:", error);
                statusMsg.innerText = "❌ Không thể kết nối tới server!";
            }
        }
    </script>

</body>
</html>
```
Giao diện trang web: web1.phuongkmt.id.vn:
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0bea44a2-1fec-42a4-9909-4e6c10e4cdcb" />
Kết quả trả về sau khi ấn lấy bảng điểm:
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/7e0374ef-c67e-4926-a6ef-eb0f15412073" />











