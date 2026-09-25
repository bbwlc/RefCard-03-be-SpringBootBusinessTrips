# EX-03 – Checkliste: ECR + ECS mit AWS CLI (AWS Academy Learner Lab)

Diese Checkliste ist die praktische Kurzform von
[EX-03-deploy-AWS-ECS.md](./EX-03-deploy-AWS-ECS.md), zugeschnitten auf einen
**AWS Academy Learner Lab**-Account: kein OIDC, keine eigenen IAM-Rollen —
stattdessen die vorgegebene `LabRole` und die temporären Session-Zugangsdaten
aus dem Lab. Jeder Schritt ist als Checkbox mit dem passenden CLI-Befehl
dokumentiert; Hintergründe/Warum stehen im Haupt-Dokument.

> Region: AWS Academy Learner Labs sind meist auf **`us-east-1`** beschränkt.
> Vor dem Start prüfen (z. B. eine Ressource testweise in einer anderen
> Region anlegen — schlägt sie mit `AuthFailure`/`UnauthorizedOperation` fehl,
> gilt die Beschränkung). Alle Befehle unten verwenden `us-east-1`; anpassen,
> falls euer Lab eine andere Region erlaubt.

---

## 0. Voraussetzungen

- [ ] Lab in AWS Academy gestartet (**Start Lab**, grüner Punkt = aktiv)
- [ ] AWS CLI lokal installiert: `aws --version`
- [ ] Profil für das Lab angelegt (siehe [EX-03-Profile.md](./EX-03-Profile.md)) mit den drei Werten aus *AWS Details → AWS CLI*:
  ```bash
  aws configure --profile academy
  # AWS Access Key ID, AWS Secret Access Key eingeben
  ```
  Der **Session Token** gehört nicht in `aws configure` — stattdessen manuell in `~/.aws/credentials` unter `[academy]` ergänzen:
  ```ini
  [academy]
  aws_access_key_id = ...
  aws_secret_access_key = ...
  aws_session_token = ...
  ```
- [ ] Profil aktivieren und Zugriff prüfen:
  ```bash
  export AWS_PROFILE=academy
  aws sts get-caller-identity
  ```
- [ ] Account-ID notieren (aus der vorigen Ausgabe, Feld `Account`) — wird unten mehrfach gebraucht:
  ```bash
  ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
  echo $ACCOUNT_ID
  ```
- [ ] Abgeschlossene EX-02 (funktionierendes `Dockerfile`, Image lässt sich lokal bauen)
- [ ] `LabRole`-ARN notieren (ersetzt in diesem Fallback die selbst angelegte IAM-Rolle):
  ```bash
  aws iam get-role --role-name LabRole --query Role.Arn --output text
  ```

---

## 1. ECR-Repository anlegen

- [ ] Repository erstellen:
  ```bash
  aws ecr create-repository --repository-name biztrips-backend --region us-east-1
  ```
- [ ] `repositoryUri` aus der Ausgabe notieren, z. B.
  `$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/biztrips-backend`

---

## 2. IAM/OIDC — **entfällt im Learner Lab**

- [ ] Nicht ausführen: `aws iam create-open-id-connect-provider` schlägt im
  Learner Lab mit `AccessDenied` fehl (kein `iam:CreateOpenIDConnectProvider`,
  kein `iam:CreateRole`). Stattdessen unten in Schritt 7 die
  Learner-Lab-Session-Credentials als GitHub Secrets verwenden.

---

## 3. RDS-Instanz für MariaDB anlegen

- [ ] Instanz erstellen:
  ```bash
  aws rds create-db-instance \
    --db-instance-identifier biztrips-backend-db \
    --db-instance-class db.t3.micro \
    --engine mariadb \
    --engine-version 10.11 \
    --master-username admin \
    --master-user-password '<sicheres-passwort>' \
    --allocated-storage 20 \
    --db-name db_biztrips \
    --publicly-accessible false \
    --region us-east-1
  ```
- [ ] Warten, bis verfügbar (dauert einige Minuten):
  ```bash
  aws rds wait db-instance-available --db-instance-identifier biztrips-backend-db --region us-east-1
  ```
- [ ] Endpoint notieren:
  ```bash
  aws rds describe-db-instances \
    --db-instance-identifier biztrips-backend-db \
    --query "DBInstances[0].Endpoint.Address" --output text --region us-east-1
  ```
- [ ] Security Group der RDS-Instanz: eingehend Port **3306** nur von der Security Group der Fargate-Tasks (`sg-tasks`, siehe Schritt 5) erlauben — nicht aus dem Internet.
- [ ] Passwort in Secrets Manager ablegen:
  ```bash
  aws secretsmanager create-secret \
    --name biztrips-backend-db-password \
    --secret-string '<dasselbe-passwort-wie-oben>' \
    --region us-east-1
  ```

---

## 4. ECS-Cluster anlegen

- [ ] Cluster erstellen:
  ```bash
  aws ecs create-cluster --cluster-name biztrips-backend-cluster --region us-east-1
  ```

---

## 5. Netzwerk: Security Groups, Application Load Balancer, Target Group

- [ ] VPC- und Subnet-IDs ermitteln (Learner Lab hat meist eine Default-VPC):
  ```bash
  aws ec2 describe-vpcs --query "Vpcs[0].VpcId" --output text --region us-east-1
  aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>" --query "Subnets[].SubnetId" --output text --region us-east-1
  ```
- [ ] Security Group `sg-alb` anlegen: eingehend Port **80** aus dem Internet (`0.0.0.0/0`)
- [ ] Security Group `sg-tasks` anlegen: eingehend Port **8080** nur von `sg-alb`; ausgehend Port **3306** zur Security Group der RDS-Instanz
- [ ] Target Group anlegen: Typ `ip`, Port **8080**, Health-Check-Pfad `/actuator/health`
  ```bash
  aws elbv2 create-target-group \
    --name biztrips-backend-tg --protocol HTTP --port 8080 \
    --vpc-id <vpc-id> --target-type ip \
    --health-check-path /actuator/health \
    --region us-east-1
  ```
- [ ] Application Load Balancer anlegen (mind. zwei Subnets, verschiedene AZs):
  ```bash
  aws elbv2 create-load-balancer \
    --name biztrips-backend-alb \
    --subnets <subnet-a> <subnet-b> \
    --security-groups <sg-alb-id> \
    --region us-east-1
  ```
- [ ] Listener auf Port 80 → Target Group anlegen:
  ```bash
  aws elbv2 create-listener \
    --load-balancer-arn <alb-arn> \
    --protocol HTTP --port 80 \
    --default-actions Type=forward,TargetGroupArn=<target-group-arn> \
    --region us-east-1
  ```

---

## 6. Task Definition anpassen und registrieren

- [ ] `task-definition.json` im Projektroot anpassen:
  - `executionRoleArn` → `arn:aws:iam::<ACCOUNT_ID>:role/LabRole` (statt einer selbst angelegten `ecsTaskExecutionRole` — im Learner Lab gibt es nur `LabRole`, die bereits ausreichend Rechte für ECR-Pull, CloudWatch-Logs und `secretsmanager:GetSecretValue` mitbringt)
  - `image` → `$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/biztrips-backend:latest`
  - `SPRING_DATASOURCE_URL` → `jdbc:mariadb://<rds-endpoint aus Schritt 3>:3306/db_biztrips`
  - `secrets[0].valueFrom` → `arn:aws:secretsmanager:us-east-1:<ACCOUNT_ID>:secret:biztrips-backend-db-password`
  - `logConfiguration.options.awslogs-region` → `us-east-1`
- [ ] Log-Group vorab anlegen (sonst schlägt der Task-Start fehl):
  ```bash
  aws logs create-log-group --log-group-name /ecs/biztrips-backend --region us-east-1
  ```
- [ ] Task Definition registrieren:
  ```bash
  aws ecs register-task-definition --cli-input-json file://task-definition.json --region us-east-1
  ```

---

## 7. ECS-Service anlegen

- [ ] Service erstellen (Platzhalter `<subnet-...>`, `<sg-tasks-id>`, `<target-group-arn>` einsetzen):
  ```bash
  aws ecs create-service \
    --cluster biztrips-backend-cluster \
    --service-name biztrips-backend-service \
    --task-definition biztrips-backend \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[<subnet-a>,<subnet-b>],securityGroups=[<sg-tasks-id>],assignPublicIp=ENABLED}" \
    --load-balancers "targetGroupArn=<target-group-arn>,containerName=biztrips-backend,containerPort=8080" \
    --region us-east-1
  ```
- [ ] Status prüfen, bis `runningCount == desiredCount`:
  ```bash
  aws ecs describe-services --cluster biztrips-backend-cluster --services biztrips-backend-service --region us-east-1
  ```

---

## 8. GitHub Actions: Learner-Lab-Credentials statt OIDC

- [ ] `deploy-ecs`-Job in `.github/workflows/deploy.yml` nutzt bereits die
  Learner-Lab-Variante (`aws-access-key-id`/`aws-secret-access-key`/
  `aws-session-token` statt `role-to-assume`) — Region dort ggf. auf
  `us-east-1` anpassen, falls das Lab andere Regionen sperrt.
- [ ] GitHub Secrets aus *AWS Details → AWS CLI* setzen (repo-level, wie `DOCKERHUB_*`):
  ```bash
  gh secret set AWS_ACCESS_KEY_ID
  gh secret set AWS_SECRET_ACCESS_KEY
  gh secret set AWS_SESSION_TOKEN
  ```
- [ ] Merken: Diese drei Werte laufen mit der Lab-Sitzung ab — bei jeder neuen
  Sitzung (bzw. spätestens nach Ablauf) müssen die drei `gh secret set`-Befehle
  wiederholt werden, sonst schlägt `configure-aws-credentials` im Workflow
  mit `ExpiredToken`/`InvalidClientTokenId` fehl.

---

## 9. Pipeline auslösen und verifizieren

- [ ] Push auf `main` (oder `gh workflow run deploy.yml`)
- [ ] Job `deploy-ecs` in GitHub Actions beobachten (`gh run watch`)
- [ ] Bei Erfolg: ALB-DNS-Name ermitteln und Health-Check testen:
  ```bash
  aws elbv2 describe-load-balancers --names biztrips-backend-alb --query "LoadBalancers[0].DNSName" --output text --region us-east-1
  curl http://<alb-dns-name>/actuator/health
  ```

---

## Aufräumen (wichtig — Learner Lab hat Kostenlimit)

- [ ] `aws ecs update-service --cluster biztrips-backend-cluster --service biztrips-backend-service --desired-count 0 --region us-east-1`
- [ ] `aws ecs delete-service --cluster biztrips-backend-cluster --service biztrips-backend-service --region us-east-1`
- [ ] `aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn> --region us-east-1`
- [ ] `aws elbv2 delete-target-group --target-group-arn <target-group-arn> --region us-east-1`
- [ ] `aws ecs delete-cluster --cluster biztrips-backend-cluster --region us-east-1`
- [ ] `aws rds delete-db-instance --db-instance-identifier biztrips-backend-db --skip-final-snapshot --region us-east-1`
- [ ] `aws ecr delete-repository --repository-name biztrips-backend --force --region us-east-1`
- [ ] `aws secretsmanager delete-secret --secret-id biztrips-backend-db-password --force-delete-without-recovery --region us-east-1`
