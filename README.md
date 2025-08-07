## 概要

MLOps の Cloudfromation の実現に向けて

## システム構成図

下記は MLOps の一部
![システム構成図](./assets/system_image.jpg)

## memo

alb_ec2sample.yaml は ACM に外部証明書か、AWS で発行した無料の証明書を発行している前提。
alb_ec2sample.yaml でスタックは成功するが、ヘルスチェックに引っかかる  
Nginx でヘルスチェックルートを作成する必要がありそう。  
後は Docker で Flask の API を持ってこれば行けそう。
下記を追加して検証したい。

```
# ターゲットグループ（ヘルスチェックパスを/healthに変更）
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    DependsOn:
      - EC2Instance
    Properties:
      Name: !Sub "${AWS::StackName}-tg"
      Protocol: HTTP
      Port: 80
      VpcId: !Ref VpcId
      TargetType: instance
      HealthCheckProtocol: HTTP
      HealthCheckPath: /health  # ← ここを変更
      Matcher:
        HttpCode: "200"
      Targets:
        - Id: !Ref EC2Instance
          Port: 80

  # EC2 インスタンス（シンプルなNginx + ヘルスチェック）
  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !Ref AmiId
      KeyName: !Ref KeyName
      NetworkInterfaces:
        - AssociatePublicIpAddress: true
          DeviceIndex: 0
          SubnetId: !Select [0, !Ref SubnetIds]
          GroupSet:
            - !Ref InstanceSecurityGroup
      Tags:
        - Key: Name
          Value: !Ref Ec2Name
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          apt install -y nginx
          cat <<'EOF' > /etc/nginx/sites-available/default
          server {
              listen 80;
              server_name _;

              client_max_body_size 5M;

              location /health {
                  return 200 'healthy';
                  add_header Content-Type text/plain;
              }

              location / {
                  proxy_pass http://127.0.0.1:5000;
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              }
          }
          EOF

          # nginx 再起動
          systemctl restart nginx
      Tags:
        - Key: Name
          Value: !Ref Ec2Name

```
