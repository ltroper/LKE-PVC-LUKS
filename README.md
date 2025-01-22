# **Deploying a LUKS-Encrypted Volume in LKE with PVC**

## **1. What is LUKS Encryption and Why Use It?**
Linux Unified Key Setup (LUKS) is a disk encryption specification that ensures data at rest is protected. Using LUKS in Kubernetes means:

- **Data at rest is protected**—if someone gains access to the block storage, they cannot read the raw data without the encryption key.
- **Transparent to applications**—inside the pod, the volume behaves like any other filesystem.
- **Security best practices**—especially useful for regulatory compliance, such as HIPAA or GDPR.

---

## **2. Setting Up the LUKS-Encrypted Storage in LKE**

### **Step 1: Create a Kubernetes Secret for the LUKS Key**
Kubernetes Secrets securely store sensitive information, such as encryption keys. The key is later used by the CSI (Container Storage Interface) driver to encrypt/decrypt the volume.

#### **secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-encrypt-example-luks-key
stringData:
  luksKey: "SECRETGOESHERE"
```

Replace `"SECRETGOESHERE"` with a strong, randomly generated key.

**Explanation:**
- `stringData` stores the LUKS key.
- The key will be referenced in the StorageClass.

Apply the Secret:
```sh
kubectl apply -f secret.yaml
```

---

### **Step 2: Define the StorageClass for LUKS-Encrypted Volumes**
A StorageClass in Kubernetes defines how storage should be provisioned.

#### **storageclass.yaml**
```yaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: linode-block-storage-retain-luks
provisioner: linodebs.csi.linode.com
reclaimPolicy: Retain
parameters:
  linodebs.csi.linode.com/luks-encrypted: "true"
  linodebs.csi.linode.com/luks-cipher: "aes-xts-plain64"
  linodebs.csi.linode.com/luks-key-size: "512"
  csi.storage.k8s.io/node-stage-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-encrypt-example-luks-key
```

**Explanation:**
- `linodebs.csi.linode.com/luks-encrypted: "true"` → Enables LUKS encryption.
- `linodebs.csi.linode.com/luks-cipher: "aes-xts-plain64"` → Uses AES encryption in XTS mode.
- `linodebs.csi.linode.com/luks-key-size: "512"` → Uses a 512-bit encryption key.
- `csi.storage.k8s.io/node-stage-secret-namespace: default` → Specifies where the Secret is stored.
- `csi.storage.k8s.io/node-stage-secret-name: csi-encrypt-example-luks-key` → References the LUKS key Secret.

Apply the StorageClass:
```sh
kubectl apply -f storageclass.yaml
```

---

### **Step 3: Create a PersistentVolumeClaim (PVC)**
A PVC requests storage from the StorageClass.

#### **pvc.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-example-pvcluks
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: linode-block-storage-retain-luks
```

**Explanation:**
- `accessModes: ReadWriteOnce` → Allows one node to mount the volume.
- `resources.requests.storage: 10Gi` → Requests 10GB of encrypted storage.
- `storageClassName: linode-block-storage-retain-luks` → Uses the encrypted StorageClass.

Apply the PVC:
```sh
kubectl apply -f pvc.yaml
```

---

### **Step 4: Deploy a Pod That Uses the Encrypted PVC**
The pod mounts the encrypted volume and allows verification of the encryption.

#### **pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: luks-encryption-test
spec:
  containers:
  - name: luks-check
    image: ubuntu
    command: ["/bin/sh", "-c", "sleep infinity"]
    securityContext:
     privileged: true
    volumeMounts:
    - mountPath: /mnt/encrypted-volume
      name: luks-volume
  volumes:
  - name: luks-volume
    persistentVolumeClaim:
      claimName: csi-example-pvcluks
```

**Explanation:**
- Uses an Ubuntu container to check encryption status.
- Runs in privileged mode to allow access to raw block devices.
- Mounts the encrypted volume at `/mnt/encrypted-volume`.

Apply the Pod:
```sh
kubectl apply -f pod.yaml
```

---

## **3. Verifying the Encryption**
Once the pod is running, check if the volume is encrypted.

### **Step 1: Enter the Pod**
```sh
kubectl exec -it luks-encryption-test -- bash
```

### **Step 2: Install Cryptsetup**
```sh
apt-get update && apt-get install -y cryptsetup
```

### **Step 3: Check Block Devices**
```sh
lsblk -f
```
This lists all block devices, including the encrypted volume.

### **Step 4: Verify LUKS Status**
```sh
cryptsetup status /dev/mapper/[pvc-name]
```
Replace `[pvc-name]` with the actual mapped device name.

### **Step 5: Examine the LUKS Header**
```sh
dmsetup table /dev/mapper/[pvc-name]
```
This will show encryption details.

---

## **4. Understanding LUKS Behavior in Kubernetes**
- **Where is the encryption happening?**  
  - The encryption occurs on the block storage level, before the filesystem is mounted.
- **Why is the data accessible inside the pod?**  
  - The CSI driver decrypts the volume before mounting it to the pod, making the encryption **transparent**.
- **How is the encryption key managed?**  
  - The key is stored securely in a Kubernetes Secret and injected by the CSI driver when the volume is mounted.

---

## **5. Security Considerations**
- **Key Management**: Use external secret management solutions like HashiCorp Vault if possible.
- **Pod Privileges**: Avoid running privileged pods in production.
- **LUKS Header Exposure**: Ensure that untrusted users cannot access `/dev/mapper/` devices.

---

## **Conclusion**
This guide walked through setting up LUKS-encrypted storage in LKE, ensuring data at rest is secure while keeping storage usage transparent for applications. Encryption is handled by the Linode CSI driver, with keys securely stored in Kubernetes Secrets.

Let me know if you need any modifications or additional details! 🚀
