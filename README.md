\# BÀI 3: MULTI-STAGE BUILD VÀ LAYER CACHING



\## 1. Mục tiêu



\* Tìm hiểu sự khác nhau giữa Single-stage Build và Multi-stage Build.

\* Áp dụng Multi-stage Build để xây dựng Docker image.

\* Tìm hiểu cơ chế Layer Caching của Docker.

\* So sánh kích thước và số lượng layer của hai image.

\* Kiểm tra ứng dụng Node.js sau khi chạy bằng container.



\## 2. Cấu trúc thư mục



```text

ex3/

├── Dockerfile

├── Dockerfile.single

├── package.json

├── server.js

└── README.md

```



\## 3. Nội dung thực hiện



\### 3.1. Chuẩn bị mã nguồn



Sử dụng lại ứng dụng Node.js Express từ bài 2, bao gồm:



\* `package.json`: khai báo thông tin dự án và thư viện phụ thuộc.

\* `server.js`: mã nguồn xử lý HTTP request.



\### 3.2. Xây dựng Single-stage Docker image



File sử dụng: `Dockerfile.single`



Các bước thực hiện:



1\. Sử dụng image nền `node:18-alpine`.

2\. Thiết lập thư mục làm việc `/app`.

3\. Sao chép file khai báo thư viện vào container.

4\. Chạy `npm install --omit=dev` để cài đặt các thư viện cần thiết.

5\. Sao chép mã nguồn ứng dụng.

6\. Mở cổng 3000 và chạy ứng dụng bằng Node.js.



Lệnh build:



```powershell

docker build -f Dockerfile.single -t hw-backend:single .

```



\### 3.3. Xây dựng Multi-stage Docker image



File sử dụng: `Dockerfile`



Dockerfile được chia thành hai stage:



\*\*Stage 1 - Builder\*\*



\* Sử dụng image `node:18-alpine`.

\* Sao chép `package.json` và các file manifest phù hợp.

\* Cài đặt thư viện bằng `npm install --omit=dev`.

\* Sao chép file `server.js`.



\*\*Stage 2 - Runtime\*\*



\* Sử dụng image `node:18-alpine`.

\* Chỉ sao chép `package.json`, `node\_modules` và `server.js` từ stage builder.

\* Mở cổng 3000 và chạy ứng dụng.



Lệnh build:



```powershell

docker build -t hw-backend:multistage .

```



Multi-stage Build cho phép tách các bước chuẩn bị ứng dụng khỏi stage chạy cuối cùng. Tuy nhiên, trong bài này, cả hai stage đều sử dụng Node.js Alpine và stage runtime vẫn cần các thư viện trong `node\_modules`.



\### 3.4. Kiểm tra kích thước image



Lệnh sử dụng:



```powershell

docker images --format "table {{.Repository}}`t{{.Tag}}`t{{.Size}}" | Select-String "hw-backend"

```



Kết quả thực tế:



| Image                   | Kích thước |

| ----------------------- | ---------: |

| `hw-backend:single`     |     196 MB |

| `hw-backend:multistage` |     186 MB |



Multi-stage image giảm 10 MB so với Single-stage image.



Tỷ lệ giảm kích thước:



```text

(196 - 186) / 196 x 100% ≈ 5,1%

```



Kết quả cho thấy Multi-stage image nhỏ hơn khoảng 5,1%. Mức giảm phụ thuộc vào image nền, thư viện phụ thuộc và các file được sao chép vào stage cuối cùng.



\### 3.5. Kiểm tra số lượng layer



Lệnh kiểm tra:



```powershell

docker image inspect hw-backend:single --format "Single-stage layers: {{len .RootFS.Layers}}"



docker image inspect hw-backend:multistage --format "Multi-stage layers: {{len .RootFS.Layers}}"

```



Kết quả thực tế:



| Image                   | Số layer |

| ----------------------- | -------: |

| `hw-backend:single`     |        8 |

| `hw-backend:multistage` |        8 |



Hai image có cùng số layer theo kết quả kiểm tra. Multi-stage Build không nhất thiết làm giảm số layer; lợi ích chính là cho phép kiểm soát các thành phần được đưa vào image cuối cùng.



\### 3.6. Kiểm tra ứng dụng Multi-stage



Khởi chạy container:



```powershell

docker run -d --name app-multistage -p 8081:3000 hw-backend:multistage

```



Kiểm tra container:



```powershell

docker ps

```



Kết quả thực tế: container `app-multistage` ở trạng thái `Up`, ánh xạ cổng máy host `8081` tới cổng container `3000`.



Kiểm tra log:



```powershell

docker logs app-multistage

```



Kết quả:



```text

Server listening on port 3000

```



Gửi HTTP request:



```powershell

curl.exe -i http://localhost:8081/

```



Kết quả nhận được:



```text

HTTP/1.1 200 OK

Content-Type: application/json; charset=utf-8



{"status":"success","message":"Hello from Docker Single-stage!"}

```



Kết quả xác nhận ứng dụng hoạt động bình thường và trả về HTTP 200.



\## 4. Tìm hiểu Layer Caching



Docker lưu lại kết quả của các bước build để có thể sử dụng lại khi những đầu vào liên quan không thay đổi.



Trong hai Dockerfile, các file khai báo thư viện được sao chép trước khi chạy lệnh cài đặt:



```dockerfile

COPY package\*.json ./

RUN npm install --omit=dev

```



Sau đó mới sao chép mã nguồn ứng dụng. Khi chỉ thay đổi `server.js` mà không thay đổi các file manifest, Docker có thể tái sử dụng cache của bước cài đặt thư viện.



Cách sắp xếp này giúp giảm thời gian build lại ứng dụng khi mã nguồn thay đổi nhưng các thư viện phụ thuộc vẫn giữ nguyên.



\## 5. Nhận xét và kết luận



Qua bài thực hành, em đã xây dựng thành công hai Docker image sử dụng Single-stage Build và Multi-stage Build.



Kết quả kiểm tra cho thấy:



\* Single-stage image có kích thước 196 MB.

\* Multi-stage image có kích thước 186 MB.

\* Cả hai image đều có 8 layer theo kết quả `docker image inspect`.

\* Container Multi-stage chạy thành công và trả về HTTP 200 khi gửi request.

\* Layer Caching giúp tái sử dụng các bước build khi đầu vào không thay đổi.



Trong bài thực hành này, mức giảm kích thước là khoảng 5,1%, thấp hơn mức kỳ vọng 50–70%. Nguyên nhân là ứng dụng nhỏ, cả hai image sử dụng cùng image nền Node.js Alpine và đều cần các thư viện để chạy. Multi-stage Build sẽ mang lại lợi ích rõ rệt hơn khi có những công cụ hoặc thành phần chỉ cần trong quá trình build và không cần trong môi trường runtime.



