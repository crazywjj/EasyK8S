

# CKA

# 考试相关信息

- 总共17道题目，考试时间2小时，每道题目的分值不同，根据题目的难易程度。满分100分，通过分数为66分。（2021年3月）
- 考纲参考：curriculum ，题目均为实操题。
- 报名方式，登录linux foundation（有国内版：https://training.linuxfoundation.cn/ ）进行报名，报名费用为300美金，当然有时候会有活动减价。购买成功后会有教程说明如何激活考试和预约考试，预约考试时，网站会提供环境检查，包括扫描你的浏览器配置，摄像头等等。购买一年内均可以预约，有一次的补考机会。
- 考试时浏览器会有一个tag是考试界面，这时我们只被允许再打开另外一个tag，且只能访问以下其中一个：
  - https://kubernetes.io/docs/home/
  - https://github.com/kubernetes
  - https://kubernetes.io/blog/
- 考试开始前监考官会检查你的考试环境，整体还是比较严格的，建议考试地点要找一个安静，且桌面干净的房间。如果你报名的是CKA-CN，也即系中文监考官的考试，那只需要带上身份证即可。



# 考试准备资源

- kubernetes 这个无可置疑，kubernetes的官方文档，里面虽也有中文翻译，但事实不怎么样，会有很多误导性的翻译，如果比较偏向看中文文档，可以看看以下这个网站：https://kuboard.cn/learning/，个人觉得也写的不错，不过还是建议大家结合英文文档一起看。

- Kubernetes-Certified-Administrator（https://github.com/walidshaari/Kubernetes-Certified-Administrator） 这个人整合了许多关于CKA的资源，且非常具有参考性，强烈推荐给大家。
- certified-kubernetes-administrator-with-practice-tests （https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/）该链接也是上面链接里面有提及到的，它是一个课程，非常友好的从零开始介绍Kubernetes，并且当一部分知识介绍完毕后，会提供一个Kubernetes cluster给我们进行练习，非常推荐给大家，当然它有个不好的地方就是没有中文字幕，如果英语比较吃力的同学请三思。
  











# 2021年考试真题

## 2021年CKA考试真题（二）

本章考点

- RBAC，参考：RBAC
- NetworkPolicy，参考：NetworkPolicy
- Voulme，参考：volumes



（一）`Create a service account name dev-sa in default namespace, dev-sa can create below components in dev namespace:`

- Deployment
- StatefulSet
- DaemonSet



**解题思路**

本题目考测RBAC，题意是创建一个service account，该service account具有在命名空间dev下创建Deployment, StatefulSet, DaemonSet的权限。我们首先应创建符合题目要求的service account以及role，然后再通过role binding进行授予权限，注意题目提及到两个命名空间。

```bash
创建名为dev-sa的service account
$ kubectl create sa dev-sa -n default

接下来给该SA赋予指定权限

创建能create 以上三个组件的role，在dev的namespace下
$ kubectl create role sa-role -n dev \
—resource=deployment,statefulset,daemonset —verb=create

创建role binding，授予service account权限
$ kubectl create rolebinding sa-rolebinding -n dev \
—role=sa-role —serviceaccount=default:dev-sa

最后可以通过以下命令，来验证是否成功。如果返回yes，则证明可以，否则反之。

$ kubectl auth can-i create deployment -n dev \
—as=system:serviceaccount:default:dev-sa

```









