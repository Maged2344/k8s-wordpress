# 🌐 EKS WordPress Deployment Guide 🖥️

This project guides you through deploying **WordPress** with **MySQL** on an **Amazon EKS** cluster. The setup includes using an **EBS volume** for persistent storage, configuring a **subdomain** to access WordPress, securing it with **SSL**, and deploying the MySQL database on an **EC2 instance**.

---

## 🚀 Steps to Setup

### 1️⃣ **Create EKS Cluster with 1 Backend Machine (t3.medium)**

- Install `eksctl`:
    ```bash
    brew install eksctl
    ```

- Create the EKS cluster:
    ```bash
    eksctl create cluster \
      --name my-cluster \
      --region us-east-1 \
      --nodegroup-name standard-workers \
      --node-type t3.medium \
      --nodes 1 \
      --nodes-min 1 \
      --nodes-max 3 \
      --managed
    ```

- Verify the cluster:
    ```bash
    kubectl get nodes
    ```

---

### 2️⃣ **Deploy WordPress and MySQL on EKS**

1. **Persistent Volume (PV) and Persistent Volume Claim (PVC) for WordPress:**

    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: wordpress-pv
    spec:
      capacity:
        storage: 1Gi
      volumeMode: Filesystem
      accessModes:
        - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: ebs-sc
      csi:
        driver: ebs.csi.aws.com
        volumeHandle: vol-0b725c7237ad80f76
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: wordpress-pvc
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: ebs-sc
    ```

2. **WordPress Deployment with MySQL Connection:**

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: wordpress
      labels:
        app: wordpress
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: wordpress
      template:
        metadata:
          labels:
            app: wordpress
        spec:
          containers:
            - name: wordpress
              image: wordpress:latest
              env:
                - name: WORDPRESS_DB_HOST
                  value: "10.0.4.155:3306"
                - name: WORDPRESS_DB_USER
                  value: "wp_user"
                - name: WORDPRESS_DB_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mysql-pass
                      key: password
                - name: WORDPRESS_DB_NAME
                  value: "wordpress_db"
              ports:
                - containerPort: 80
              volumeMounts:
                - name: wordpress-storage
                  mountPath: /var/www/html
          volumes:
            - name: wordpress-storage
              persistentVolumeClaim:
                claimName: wordpress-pvc
    ```

3. **WordPress Service:**

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: wordpress
      labels:
        app: wordpress
    spec:
      ports:
        - nodePort: 30822
          port: 80
          targetPort: 80
      selector:
        app: wordpress
      type: ClusterIP
    ```

4. **MySQL Secret:**

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: mysql-pass
    type: Opaque
    data:
      password: bWFnZWQ1MDA=  # Base64 encoded value of 'mage500'
    ```

---

### 3️⃣ **Configure Subdomain for WordPress**

1. Expose the WordPress service using LoadBalancer:
    ```bash
    kubectl expose deployment wordpress --type=LoadBalancer --name=wordpress-lb
    ```

2. Get the external IP of the service:
    ```bash
    kubectl get svc wordpress-lb
    ```

3. Update DNS to point your subdomain (`yourname-wordpress.cloud-stacks.com`) to the LoadBalancer IP.

---

### 4️⃣ **Configure SSL for WordPress**

1. Install **cert-manager** for SSL certificates:
    ```bash
    kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
    ```

2. Create an **Issuer** for Let’s Encrypt:
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: your-email@example.com
        privateKeySecretRef:
          name: letsencrypt-prod
        solvers:
          - http01:
              ingress:
                class: nginx
    ```

3. Create a **Certificate**:
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: wordpress-cert
    spec:
      secretName: wordpress-tls
      dnsNames:
        - yourname-wordpress.cloud-stacks.com
      issuerRef:
        name: letsencrypt-prod
        kind: Issuer
    ```

---

### 5️⃣ **Deploy MySQL Database on EC2 Instance**

1. Launch an EC2 instance (`t3.medium`) with a security group allowing MySQL traffic (port 3306).

2. Install MySQL on EC2:
    ```bash
    sudo apt update
    sudo apt install mysql-server
    ```

3. Create a database and user for WordPress:
    ```bash
    sudo mysql -u root
    CREATE DATABASE wordpress_db;
    CREATE USER 'wp_user'@'%' IDENTIFIED BY 'wp_password';
    GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'%';
    FLUSH PRIVILEGES;
    ```

4. Update WordPress configuration to point to the EC2 MySQL instance.

---

### 6️⃣ **Optional: Manage Your Cluster with Lens**

1. Install **Lens** from [Lens](https://k8slens.dev/).
2. Connect to your EKS cluster using the kubeconfig file.

---

### 📜 **Conclusion**

Congrats! 🎉 You've successfully set up:

- **EKS cluster** with **WordPress** and **MySQL**
- **SSL** via **Let’s Encrypt**
- A **subdomain** for accessing WordPress

You can access your WordPress site securely at: `https://yourname-wordpress.cloud-stacks.com`.

---

### 📚 **References:**

- [Kubernetes WordPress MySQL Persistent Volume Tutorial](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)
- [Let’s Encrypt SSL Setup](https://cert-manager.io/docs/)
- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)

---

Happy deploying! 🚀
