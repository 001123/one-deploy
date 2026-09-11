# Argo CD trên OpenNebula dev

Role triển khai Argo CD bằng Helm, tích hợp CMP SOPS/Age cho repository
`gitops-opennebula`. Phiên bản mặc định: chart `argo-cd` **10.8.4**, Argo CD
**v3.5.2**, được đối chiếu với release chính thức ngày 2026-09-11.

## Cài đặt hoặc nâng cấp

Chạy từ thư mục `one-deploy`, dùng rõ kubeconfig của cụm dev:

```bash
.venv/bin/ansible-playbook -i localhost, playbooks/argocd.yml \
  -e argocd_kubeconfig="$HOME/.kube/config-opennebula-dev" \
  -e argocd_context=default
```

Argo CD được quản lý bởi release Helm `argo-cd` trong namespace `argocd`.
SOPS CMP dùng biểu thức Helm để lấy đúng image của repo-server, kể cả khi
override image repository/tag; không ghim một phiên bản CMP riêng.

Role mặc định giữ `application.resourceTrackingMethod=label` để nâng cấp từ
Argo CD 2.x mà không thay đổi cách nhận diện tài nguyên đang được quản lý.
Label key giữ theo chart: `argocd.argoproj.io/instance`. Có thể đặt
`argocd_resource_tracking_method=annotation` khi thực hiện một đợt migration
tracking riêng, kèm full sync các Application.

NodePort vẫn là HTTP `30080`, HTTPS `30443`. Phiên bản SOPS `3.9.4` và khóa Age hiện
có được giữ nguyên. Playbook không in private key hoặc mật khẩu admin ra log.

## Kiểm tra trước và sau nâng cấp

- Đọc hướng dẫn nâng cấp cho toàn bộ các minor version được đi qua. Với cụm dev
  hiện tại, không dùng SSO/RBAC tùy chỉnh, legacy repository trong ConfigMap,
  ApplicationSet, source hydration, UI extension hoặc registry OCI HTTP.
- Sao lưu cấu hình, Application/AppProject, Secret, CRD, Helm values và manifest
  trước nâng cấp; mã hóa backup bằng SOPS/Age, không commit bản rõ hoặc private key.
- Render/lint chart với Kubernetes `1.34.2`, kiểm tra image CMP khớp repo-server,
  Service NodePort và selector bất biến của Deployment/StatefulSet.
- Helm quản lý và nâng cấp CRD trong templates; không áp dụng ApplicationSet CRD
  lớn bằng client-side `kubectl apply`.
- Chart 10 bật NetworkPolicy mặc định; nghiệm thu kết nối giữa các thành phần,
  đăng nhập API/UI và hard refresh các Application để kiểm tra render Helm/SOPS.
- Sau nâng cấp, `root-application`, `nginx-demo`, `monitoring` phải
  `Synced/Healthy`; dữ liệu Secret và cấu hình ứng dụng được giữ nguyên.

```bash
kdev() {
  kubectl --kubeconfig="$HOME/.kube/config-opennebula-dev" --context=default "$@"
}
kdev -n argocd get pods,applications
kdev -n argocd annotate application root-application nginx-demo monitoring \
  argocd.argoproj.io/refresh=hard --overwrite
kdev -n argocd port-forward svc/argo-cd-argocd-server 8443:443
```

Mở `https://localhost:8443`, username `admin`. Lấy mật khẩu khởi tạo tại terminal
khác nếu tài khoản chưa đổi mật khẩu:

```bash
kdev -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 --decode
echo
```

## Khôi phục nếu nâng cấp thất bại

Xem `helm history argo-cd -n argocd` với đúng kubeconfig/context và chọn revision
thành công ngay trước nâng cấp. Dùng `helm rollback argo-cd <revision> --wait
--timeout 10m -n argocd` với cùng kubeconfig/context. Sau đó kiểm tra lại CRD,
Secret, Application và khả năng giải mã SOPS theo backup. Không uninstall release
hoặc xóa Application để rollback. Cập nhật chart version trong defaults tương ứng
để lần chạy playbook sau không vô tình nâng cấp lại.

Tài liệu chính thức:
- [Chart argo-cd](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd)
- [Argo CD 3.5.2](https://github.com/argoproj/argo-cd/releases/tag/v3.5.2)
- [Hướng dẫn nâng cấp](https://argo-cd.readthedocs.io/en/stable/operator-manual/upgrading/overview/)
- [Thay đổi tracking trong 3.0](https://argo-cd.readthedocs.io/en/stable/operator-manual/upgrading/2.14-3.0/#use-annotation-based-tracking-by-default)
