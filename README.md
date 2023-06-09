### Traefik

<div align="center"><img src="img/Detail.png" width="800px"></div>

- Reverse Proxy และ Load balance
- จัดการเข้าถึง service ของ docker container โดยใช้ การระบุจาก URL เพื่อเข้าถึง service ต่าง ๆ โดย URL นั้น จะทำการชี้ไปที่ IP หรือ Domain ที่ตัว Traefik ทำงานอยู่
- ลดปัญหาการใช้งาน port ที่ซ้ำกัน และการจำเลข port หากมี service ที่เยอะมาก

- service Traefik ชื่อ api@internal

<details>
<summary>SWARM</summary>

-   ไฟล์ .yaml service ของ Swarm
    ```yaml
        version: '3.3' #ต้องเวอร์ชั่นมากกว่า 3
        services: 
            api: 
                image: thanawat1303/fastapi-main:v1 
                networks: # กำหนดเครือข่ายที่ Traefik ทำงานอยู่ เพื่อให้ Traefik ทำการควบคุมได้
                    - traefik 
                environment: 
                    PORT: 8000 
                logging:
                driver: json-file 
                volumes: 
                    - /var/run/docker.sock:/var/run/docker.sock
                    - app:/app 
                restart: 'no'
                deploy: # กำหนดการ Deploy
                    replicas: 1 # กำหนดจำนวนที่ต้องการให้ไปสร้าง container เพื่อเริ่มทำงาน service
                    labels: # กำหนด Label ที่ของ container
                        - traefik.docker.network=traefik # ชื่อ network ที่ traefik ทำงานร่วมอยู่ด้วย
                        - traefik.enable=true # สั่งให้เชื่อมต่อกับ traefik
                        - traefik.http.routers.spcn19fastapi-https.entrypoints=websecure # ให้ Traefik เปลี่ยนเส้นทาง Protocal ให้เป็น websecure , https หรือ 443 เสมอ 
                        - traefik.http.routers.spcn19fastapi-https.rule=Host("spcn19fastapi.xops.ipv9.xyz") # URL ในการเข้าถึง service ของ container นั้น
                        - traefik.http.routers.spcn19fastapi-https.tls.certresolver=default # กำหนดรูปแบบการสร้าง cert
                        - traefik.http.services.spcn19fastapi.loadbalancer.server.port=8000 # กำหนดให้มีการ Load balance ตาม port ที่ service ทำงาน
                        - traefik.http.routers.spcn19fastapi-https.tls=true # เปิดใช้งาน protocal tls
                    resources: # กำหนดความต้องการของ service
                        reservations: # ค่าต่ำสุด
                            cpus: '0.1' 
                            memory: 10M
                        limits: # ค่าสูงสุด
                            cpus: '0.4'
                            memory: 250M
                
        networks: # กำหนดเครือข่ายของ service ที่สร้างผ่านไฟล์ compose นี้
            traefik: # เครือข่ายที่ traefik ทำงาน
                external: true # กำหนดให้ใช้งาน network ที่มีอยู่แล้ว
    ```

-   [ดูเพิ่มเติม](https://github.com/thanawat1303/swarm02)

</details>

<details>
<summary>K8S</summary>

-   ไฟล์ .yaml ของ K8S
    ```yaml
    --- # ส่วนในการสร้าง container และกำหนด scale ต่าง ๆ
    apiVersion: apps/v1
    kind: Deployment
    metadata:
        name: rancher-deployment
        namespace: spcn19
    spec:
        replicas: 1
        selector:
            matchLabels:
                app: rancher
        template:
            metadata:
                labels:
                    app: rancher
            spec:
                containers:
                -   name: rancher
                    image: rancher/hello-world
                    ports:
                    - containerPort: 80
    --- # ส่วนในการกำหนดจุดที่เป็น service เพื่อเข้าถึง service ภายใน container
    apiVersion: v1
    kind: Service
    metadata:
        name: rancher-service
        labels:
            name: rancher-service
        namespace: spcn19
    spec:
        selector:
            app: rancher
        ports:
        -   name: http
            port: 80 # port บน Host
            protocol: TCP
            targetPort: 80 # port ภายใน container
    --- # ส่วนในการเข้าถึง service ผ่าน Traefik
    apiVersion: traefik.containo.us/v1alpha1 # apiversion ของ Traefik
    kind: IngressRoute # Kind ของ Traefik ที่ช่วยให้กำหนดเส่นทางในการเข้า service ได้
    metadata:
        name: service-ingress
        namespace: spcn19
    spec:
        entryPoints: # กำหนด Protocal ที่สามารถเข้าถึงได้
            -   web # http
            -   websecure #https
        routes:
        -   match: Host(`web.spcn19.local`) # URL ในการเข้าถึง
            kind: Rule
            services:
            -   name: rancher-service
                port: 80
    ```

-   Kind : IngressRoute 
    -   สำหรับใช้กำหนดเส้นทางแบบขั้นสูงในการเข้าถึง service ภายใต้การทำงานของ Treafik เท่านั้น เช่น กำหนด middleware ได้

-   apiVersion: traefik.containo.us/v1alpha1
    - apiVersion ของ Traefik จัดการตั้งค่าต่างๆ ทั้งการเชื่อมต่อเครือข่ายเพื่อเข้าถึง application และกำหนดค่าต่างๆ ของ Traefik

-   [ดูเพิ่มเติม](https://github.com/thanawat1303/Kube)

</details>