---
layout: post
title: Setup VMware vSphere, vSan 
subtitle: Ảo hóa hạ tầng máy chủ và lưu trữ cho Tổ chức, Doanh nghiệp
tags: [system, vmware]
author: Anh Le
comments: false
mathjax: false
---
 
- [1. Giới thiệu](#1-giới-thiệu)
- [2. Các công nghệ sử dụng](#2-các-công-nghệ-sử-dụng)
- [3. Mô hình triển khai](#3-mô-hình-triển-khai)

### 1. Giới thiệu 

- Các ứng dụng dịch vụ đòi hỏi môi trường triển khai để hoạt động, thường là các máy chủ (server). Trong hệ thống 1 doanh nghiệp có hàng trăm, hàng ngàn dịch vụ khác nhau, do đó đòi hỏi 1 lượng lớn máy chủ. Ảo hóa hạ tầng các máy chủ vật lý cung cấp ra thành các máy chủ ảo đem đến lợi ích lớn về việc tận dụng hiệu quả, tối đa tài nguyên phần cứng máy chủ từ đó giảm chi phí đầu tư hạ tầng cho doanh nghiệp.

- Trong bài viết này mình chia sẻ về cách triển khai hệ thống ảo hóa với VMware vSphere và vSan, version 7.0 

<img src="../assets/img/2024-10-23-setup-vmware-vsphere-vsan/vmware.gif"/>


### 2. Các công nghệ sử dụng
- VMware ESXi 7.0
- VMware vCenter 7.0
- VMware vSan 7.0

### 3. Mô hình triển khai 

- sdada
