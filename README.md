# cicd-lab

Một FastAPI app có đúng ba endpoint để quan sát luồng **Mac → GitHub → CI → CD → VPS → Docker → health check**. CI chạy khi push bất kỳ branch nào; CD chỉ chạy cho `main` sau khi CI thành công. Cổng dùng trong toàn bộ bài lab là `8000`.

```text
Mac (viết code, chạy thử, git push)
  ↓
GitHub (lưu repository)
  ↓
GitHub Actions runner (chạy CI: pytest, docker build)
  ↓ CI pass trên main
GitHub Actions runner (chạy CD: lệnh SSH)
  ↓ SSH
VPS Ubuntu (git fetch, docker compose, gọi /health)
  ↓
Docker container trên VPS (chạy FastAPI)
```

## 1. Chạy app trên Mac

Trong thư mục `cicd-lab`:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
uvicorn app.main:app --host 127.0.0.1 --port 8000
```

CI và Docker dùng Python 3.13; bạn có thể dùng `python3` sẵn có trên Mac để học và chạy thử. Mở terminal khác và thử:

```bash
curl http://127.0.0.1:8000/
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/version
```

Kết quả lần lượt là `{"message":"CI/CD Lab"}`, `{"status":"ok"}`, `{"version":"v1"}`. Dừng server bằng `Ctrl+C` trước khi chạy Compose vì cả hai dùng cổng `8000`.

## 2. Chạy test trên Mac

```bash
python -m pytest -q
```

Ba test trong `tests/test_app.py` kiểm tra status code và JSON của từng endpoint. `httpx` là dependency mà `TestClient` dùng để gọi app trong test.

## 3. Chạy bằng Docker

```bash
docker compose up -d --build
curl http://127.0.0.1:8000/health
docker compose logs app
```

Để dừng: `docker compose down`. `Dockerfile` khởi động Uvicorn trên cổng `8000`; `compose.yaml` nối cổng `8000` của máy với cổng `8000` của container. Image chứa cả dependency test để chỉ cần một `requirements.txt` cho bài lab.

## 4. Tạo GitHub repo và push

Tạo một **public repository trống** tên `cicd-lab` trên GitHub, không chọn tạo sẵn README. Repo public giúp VPS chạy `git fetch` mà không cần thêm credential. Đặt `<GITHUB_USER>` trong các lệnh bên dưới thành username của bạn.

Nếu dùng repo private, hãy cấu hình quyền đọc repo cho Git trên VPS trước khi deploy; bài lab này không thêm GitHub secret cho bước đó.

Trong thư mục project trên Mac:

```bash
git init -b main
git add .
git commit -m "Initial CI/CD lab"
git remote add origin https://github.com/<GITHUB_USER>/cicd-lab.git
```

**Chuẩn bị VPS và Secrets ở mục 6–7 trước khi push lần đầu.** Sau đó:

```bash
git push -u origin main
```

Mỗi lần sửa tiếp theo: `git add .`, `git commit -m "..."`, `git push`.

## 5. CI chạy ở đâu và làm gì?

File `.github/workflows/ci-cd.yml` chạy trên **GitHub Actions runner (Ubuntu)** khi có push. Job `ci` lần lượt checkout commit được push, cài Python 3.13, cài `requirements.txt`, chạy `python -m pytest -q`, rồi chạy `docker build -t cicd-lab:ci .`. Bước nào lỗi thì job `ci` fail; job `cd` phụ thuộc vào `ci` nên bị skip. Docker image build trong CI chỉ để kiểm tra khả năng build; VPS tự build lại từ source khi deploy.

## 6. Chuẩn bị VPS Ubuntu

VPS cần sẵn Git, Docker Engine, Docker Compose plugin và user deploy chạy được `docker`. Health check còn dùng `curl`; nếu chưa có, cài bằng `sudo apt update && sudo apt install -y curl`. VPS cần kết nối được GitHub để `git fetch` và kết nối được PyPI/Docker Hub để build image.

Sau khi tạo GitHub repo trống ở mục 4, SSH vào VPS bằng user deploy và tạo checkout ở đúng đường dẫn:

```bash
sudo mkdir -p /opt/cicd-lab
sudo chown "$USER":"$USER" /opt/cicd-lab
cd /opt/cicd-lab
git init -b main
git remote add origin https://github.com/<GITHUB_USER>/cicd-lab.git
```

Checkout này chưa có commit; lần CD đầu sẽ `git fetch` và lấy commit vừa qua CI. Nếu đã push trước khi chuẩn bị VPS, lần CD đầu sẽ fail; chuẩn bị VPS rồi push một commit mới để chạy lại.

Tạo một SSH key riêng trên Mac nếu chưa có:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/cicd-lab -N "" -C "cicd-lab deploy"
```

Key không có passphrase để Actions có thể dùng tự động. Thêm nội dung file `~/.ssh/cicd-lab.pub` vào `~/.ssh/authorized_keys` của user deploy trên VPS. Xác nhận kết nối SSH bằng key đó trước khi push. User này cũng cần sở hữu `/opt/cicd-lab` và có quyền chạy `docker compose`.

## 7. Thêm GitHub Secrets

Trong GitHub repo, vào **Settings → Secrets and variables → Actions → New repository secret** và tạo đúng ba secret:

| Secret | Giá trị |
| --- | --- |
| `VPS_HOST` | IP hoặc hostname VPS |
| `VPS_USER` | user deploy trên VPS |
| `VPS_SSH_KEY` | toàn bộ nội dung private key `~/.ssh/cicd-lab`, gồm dòng BEGIN/END |

Không commit private key vào Git. Workflow dùng cổng SSH mặc định `22` và chấp nhận host key ở lần kết nối đầu; không cần thêm secret khác cho repo public.

## 8. CD chạy như thế nào?

Job `cd` chỉ chạy khi job `ci` pass **và** branch được push là `main`. Runner dùng SSH vào VPS, `cd /opt/cicd-lab`, chạy `git fetch origin main`, rồi đặt checkout về **đúng commit SHA đã qua CI** bằng `git reset --hard`. Vì vậy, đừng sửa source trực tiếp trong checkout trên VPS; các thay đổi chưa commit ở đó sẽ bị bỏ.

Trên VPS, workflow chạy `docker compose up -d --build`, thử `http://127.0.0.1:8000/health` tối đa 10 lần (mỗi lần cách 2 giây), và chỉ báo thành công khi HTTP là 2xx **và** nội dung là `{"status":"ok"}`. Nếu thất bại, job in 50 dòng log container gần nhất rồi fail. Phiên bản này **không rollback**: khi health check fail sau khi container mới đã chạy, production có thể vẫn đang lỗi cho đến khi bạn sửa và push lại.

## 9. Xem GitHub Actions log

Vào tab **Actions** của GitHub repo → chọn workflow **CI/CD Lab** → chọn lần chạy theo commit → mở job `ci` hoặc `cd`. Nếu test hoặc Docker build fail, `ci` có dấu đỏ và `cd` hiện **Skipped**. Nếu health check fail, mở step **Deploy and check health on VPS** trong job `cd` để xem các lần thử và log container. Trên VPS cũng có thể chạy `cd /opt/cicd-lab && docker compose logs --tail=50 app`.

## 10. Kiểm tra version production

Sau khi CD xanh, chạy từ Mac (nếu cổng `8000` của VPS có thể truy cập từ Mac):

```bash
curl http://<VPS_HOST>:8000/version
```

Hoặc SSH vào VPS rồi chạy `curl http://127.0.0.1:8000/version`. Kết quả ban đầu là `{"version":"v1"}`. Nếu cổng ngoài không mở, cách qua SSH vẫn kiểm tra được mà không cần đổi firewall.

# Experiments

Thực hiện theo thứ tự, trên branch `main`, và chờ mỗi workflow kết thúc trước khi push thí nghiệm kế tiếp.

1. **Push bình thường:** Sau khi hoàn tất mục 4, 6, 7, push commit đầu. Xem `ci` pass cả pytest lẫn Docker build, rồi `cd` pass health check. Production `/version` trả `v1`.
2. **Cho pytest fail:** Trong `tests/test_app.py`, đổi assertion của `test_home` thành `assert response.json() == {"message": "sai"}`. Commit và push. Test fail trong `ci`; `cd` hiện **Skipped**. Production vẫn là bản cũ.
3. **Sửa lại test:** Đổi assertion về `{"message": "CI/CD Lab"}`, commit và push. `ci` pass, `cd` chạy và health check pass.
4. **Đổi version:** Trong `app/main.py`, đổi `"v1"` thành `"v2"` ở endpoint `/version`. Đồng thời đổi giá trị mong đợi trong `test_version` thành `"v2"`. Commit và push; khi CD xanh, gọi `/version` trên VPS để thấy `v2`. Lặp lại với `v3` nếu muốn.
5. **Cho Docker build fail:** Tạm thêm `COPY file-khong-ton-tai .` vào `Dockerfile`, commit và push. Pytest pass nhưng Docker build fail, nên `cd` bị skip. Xóa dòng đó, commit và push để phục hồi.
6. **Cho health check fail sau deploy:** Tạm sửa `/health` trong `app/main.py` để trả HTTP `503` với cùng JSON `{"status":"ok"}` (ví dụ `return JSONResponse(status_code=503, content={"status": "ok"})` và thêm `from fastapi.responses import JSONResponse`). Đổi **chỉ** assertion status code trong `test_health` từ `200` thành `503` để CI pass trong thí nghiệm này. Commit và push: CI pass, CD deploy container mới, rồi health check fail với log rõ ràng. Đây là lỗi **được phát hiện sau khi deploy**, không có rollback. Hoàn tác cả hai file, commit và push để production khỏe lại.

Các thí nghiệm 2 và 5 cho thấy CI chặn CD trước khi chạm VPS. Thí nghiệm 6 cho thấy kiểm tra sau deploy phát hiện lỗi chạy thực tế, nhưng bản lab này chưa tự phục hồi production.
