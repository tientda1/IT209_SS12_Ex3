# Bài 3: Tách biệt luồng CI/CD đa môi trường (Staging & Production) với GitHub Environments

Tách luồng triển khai thành 2 môi trường bằng tính năng **GitHub Environments**:
`staging` (tự động khi push vào `develop`) và `production` (có **Required
reviewers**, phải phê duyệt thủ công khi push vào `main`).

## Cấu trúc

```
ex3/
├── .github/
│   └── workflows/
│       └── cd-multi-env.yml   # 2 job: deploy-staging, deploy-prod
├── screenshots/
│   └── .gitkeep               # nơi đặt ảnh chụp màn hình phê duyệt
└── README.md
```

> Tệp nộp bài chính là `.github/workflows/cd-multi-env.yml` kèm **ảnh chụp màn hình
> cửa sổ phê duyệt deployment Production** (đặt trong `screenshots/`).

## Điểm chính trong `cd-multi-env.yml`

- Workflow lắng nghe `push` trên cả `develop` và `main`, nhưng mỗi job tự lọc theo
  nhánh bằng `if`:

  ```yaml
  on:
    push:
      branches: [ develop, main ]

  jobs:
    deploy-staging:
      if: github.ref == 'refs/heads/develop'   # chỉ chạy trên develop
      environment:
        name: staging
        url: https://api.staging.example.com

    deploy-prod:
      if: github.ref == 'refs/heads/main'      # chỉ chạy trên main
      environment:
        name: production
        url: https://api.prod.example.com
  ```

**Vì sao dùng từ khóa `environment`?** Đây là thứ "gắn" job với một môi trường đã
khai báo trong repo. Nhờ đó job thừa hưởng toàn bộ **Environment Protection Rules**:
required reviewers, wait timer, giới hạn nhánh được deploy... Nếu bỏ `environment`,
job vẫn chạy nhưng **không có cơ chế chờ phê duyệt**.

**Vì sao tách 2 job thay vì 2 workflow?** Gộp vào một tệp giúp nhìn được toàn bộ
luồng đa môi trường ở một chỗ và dễ so sánh; điều kiện `if` theo nhánh thay cho việc
nhân bản cấu hình `on.push.branches` ở nhiều file.

## 1. Cấu hình Environments trên GitHub

1. Vào repo → **Settings** → **Environments** → **New environment**.
2. Tạo môi trường thứ nhất tên **`staging`** (không cần bảo vệ).
3. Tạo môi trường thứ hai tên **`production`** (khớp đúng tên trong YAML:
   `environment.name: production`).
4. Trong môi trường **`production`**, bật:
   - **Required reviewers** → thêm tài khoản cá nhân của bạn làm người duyệt
     (tối đa 6 người). Có thể bật thêm **Prevent self-review** nếu muốn tách vai trò.
   - (Tuỳ chọn) **Wait timer**, **Deployment branches → Selected branches → `main`**
     để chỉ nhánh `main` được deploy lên production.

## 2. Kiểm tra luồng chờ phê duyệt

1. Push/merge code vào `main`:

   ```bash
   git push origin main
   ```

2. Vào tab **Actions** → chọn run **CD Multi-Environment** → job **Deploy to
   Production** hiển thị trạng thái **Waiting for review** (màu vàng), kèm nút
   **Review deployments**.
3. Bấm **Review deployments** → chọn `production` → **Approve and deploy**.
4. Job tiếp tục thực thi và kết thúc với **tích xanh** (success).

> Push vào `develop` sẽ chạy job **Deploy to Staging** và **không** dừng chờ duyệt,
> thể hiện đúng sự tách biệt giữa hai môi trường.

## 3. Ảnh chụp cần nộp

Chụp màn hình **cửa sổ phê duyệt deployment môi trường Production**, thể hiện rõ:

- Tên môi trường `production`.
- Thông báo **Waiting for review** / nút **Review deployments**.
- Nút **Approve and deploy** (và có thể cả trạng thái xanh sau khi duyệt).

Lưu vào `screenshots/approve-production.png`.

## Ghi chú

- `url:` trong `environment` chỉ là **liên kết hiển thị** (xuất hiện ở phần
  Deployments của repo); không ảnh hưởng tới việc chờ duyệt.
- Các bước `run` ở đây mô phỏng việc deploy (`echo`); trong dự án thật bạn thay
  bằng lệnh deploy thực tế (ks, ssh, `docker push`...).
- Bí mật như token triển khai nên khai báo trong **Environment secrets** của từng
  môi trường thay vì secrets cấp repo, để tách quyền giữa staging và production.
