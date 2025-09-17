# threads-infra
kubectl edit application ingress-nginx -n argocd => xóa ingerss

argo pass: 
```
╰─ kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo                                                                 ─╯

ibhLYtMlTn7mdpag

argocd login localhost:8080 --username admin --password ibhLYtMlTn7mdpag  
```


argo 8080
drone 8081
harbor 8082

# Không đi riêng xe vì:
- Người đi chậm ngại vì làm ae phải đi chậm theo
- Người đi nhanh thì khó chịu vì phải đi chậm
- Mỗi người 1 xe thì nhỡ trên đường có gì hay thì chụp ảnh kiểu gì
