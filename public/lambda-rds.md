---
title: 【AWS CDK】Lambda in VPCからRDSにアクセスする
tags:
  - AWS
  - RDS
  - lambda
  - CDK
private: false
updated_at: '2024-06-23T07:26:14+09:00'
id: 349cdc77fa804f10b632
organization_url_name: null
slide: false
ignorePublish: false
---

## TL;DR

とりあえずソース全文

<details>
  <summary>lib/cdk-sample-stack.ts</summary>

```typescript
import * as ec2 from "aws-cdk-lib/aws-ec2";
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as secretmanager from "aws-cdk-lib/aws-secretsmanager";
import * as rds from "aws-cdk-lib/aws-rds";
import { Construct } from "constructs";

const DB_NAME = "sampleDB";
const DB_USER = "sampleUser";
const DB_PORT = 3306;

export class CdkSampleStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // VPC
    const vpc = new ec2.Vpc(this, "SampleVPC", {
      ipAddresses: ec2.IpAddresses.cidr("10.0.0.0/16"),
      subnetConfiguration: [
        {
          cidrMask: 24,
          name: "isolatedSubnet",
          subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
        },
      ],
      maxAzs: 2,
    });
    new ec2.InterfaceVpcEndpoint(this, "SecretmanagerEndPoint", {
      vpc,
      service: ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
    });

    // RDS & Proxy
    const rdsSecret = new secretmanager.Secret(this, "RDSSecret", {
      generateSecretString: {
        generateStringKey: "password",
        secretStringTemplate: `{"username": "${DB_USER}"}`,
        excludePunctuation: true,
      },
    });
    const rdsSecurityGroup = new ec2.SecurityGroup(this, "RDSSecurityGroup", {
      vpc,
    });
    const rdsProxySecurityGroup = new ec2.SecurityGroup(
      this,
      "RDSProxySecurityGroup",
      { vpc },
    );
    const rdsInstance = new rds.DatabaseInstance(this, "SampleRDS", {
      engine: rds.DatabaseInstanceEngine.MYSQL,
      databaseName: DB_NAME,
      instanceType: ec2.InstanceType.of(
        ec2.InstanceClass.T3,
        ec2.InstanceSize.SMALL,
      ),
      vpc,
      subnetGroup: new rds.SubnetGroup(this, "RDSSubnetGroup", {
        description: "subnetGroup",
        vpc,
        vpcSubnets: vpc.selectSubnets({
          subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
        }),
      }),
      securityGroups: [rdsSecurityGroup],
      credentials: rds.Credentials.fromSecret(rdsSecret),
    });
    const rdsProxy = rdsInstance.addProxy("SampleRDSProxy", {
      secrets: [rdsSecret],
      vpc,
      securityGroups: [rdsProxySecurityGroup],
      requireTLS: false,
    });

    // Lambda
    const lambdaSecurityGroup = new ec2.SecurityGroup(
      this,
      "LambdaSecurityGroup",
      { vpc },
    );
    const libsLayer = new lambda.LayerVersion(this, "SampleLayer", {
      code: lambda.Code.fromAsset("src/libs"),
      compatibleRuntimes: [lambda.Runtime.PYTHON_3_9],
    });
    const lambdaFunction = new lambda.Function(this, "SampleLambda", {
      runtime: lambda.Runtime.PYTHON_3_9,
      code: lambda.Code.fromAsset("src/functions"),
      handler: "sample.lambda_handler",
      environment: {
        SECRET_ARN: rdsSecret.secretArn,
        PROXY_ENDPOINT: rdsProxy.endpoint,
      },
      layers: [libsLayer],
      vpc,
      securityGroups: [lambdaSecurityGroup],
    });
    rdsSecret.grantRead(lambdaFunction);
    rdsInstance.grantConnect(lambdaFunction, DB_USER);

    // SecurityGroup rules
    rdsProxySecurityGroup.addIngressRule(
      lambdaSecurityGroup,
      ec2.Port.tcp(DB_PORT),
      "Allow from lambda",
    );
    rdsSecurityGroup.addIngressRule(
      rdsProxySecurityGroup,
      ec2.Port.tcp(DB_PORT),
      "Allow from RDSProxy",
    );
  }
}
```

</details>

<details>
  <summary>src/functions/sample.py</summary>

```python
import json
import logging
import os
import sys

import boto3
import pymysql

# get secrets
secret_manager = boto3.client("secretsmanager")
secret_arn = os.getenv("SECRET_ARN")
secret = secret_manager.get_secret_value(SecretId=secret_arn)
secret_values = json.loads(secret["SecretString"])

# credentials
user_name = secret_values["username"]
password = secret_values["password"]
rds_proxy_host = os.getenv("PROXY_ENDPOINT")
db_name = secret_values["dbname"]

logger = logging.getLogger()
logger.setLevel(logging.INFO)

try:
    conn = pymysql.connect(
        host=rds_proxy_host,
        user=user_name,
        passwd=password,
        db=db_name,
        connect_timeout=5,
    )
except pymysql.MySQLError as e:
    logger.error("ERROR: Unexpected error: Could not connect to MySQL instance.")
    logger.error(e)
    sys.exit(1)

logger.info("SUCCESS: Connection to RDS for MySQL instance succeeded")


def lambda_handler(event, context):
    cust_id = "1"
    name = "custNameSample"

    sql_string = f"insert into Customer (CustID, Name) values(%s, %s)"

    with conn.cursor() as cur:
        cur.execute(
            "create table if not exists Customer ( CustID  int NOT NULL, Name varchar(255) NOT NULL, PRIMARY KEY (CustID))"
        )
        cur.execute(sql_string, (cust_id, name))
        conn.commit()

        cur.execute("select * from Customer")
        result = cur.fetchall()
        logger.info(f"{result=}")
    conn.commit()
```

</details>

lambdaの実装は[公式のサンプル](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/rds-lambda-tutorial.html)から借用しました。

## VPC

```typescript
const vpc = new ec2.Vpc(this, "SampleVPC", {
  ipAddresses: ec2.IpAddresses.cidr("10.0.0.0/16"),
  subnetConfiguration: [
    {
      cidrMask: 24,
      name: "isolatedSubnet",
      subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
    },
  ],
  maxAzs: 2, // RDSProxyはsubnetが2つ必要
});
new ec2.InterfaceVpcEndpoint(this, "SecretmanagerEndPoint", {
  vpc,
  service: ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
});
```

- RDSProxyで指定するサブネットグループは2つ以上のサブネットを含む必要があります。
- 今回RDSのクレデンシャルをsecretsManagerで管理するので、VPCにエンドポイントをアタッチします。

## RDS & RDSProxy

```typescript
const rdsSecret = new secretmanager.Secret(this, "RDSSecret", {
  generateSecretString: {
    generateStringKey: "password",
    secretStringTemplate: `{"username": "${DB_USER}"}`,
    excludePunctuation: true, // 自動生成されるパスワードから句読点を除外
  },
});
const rdsSecurityGroup = new ec2.SecurityGroup(this, "RDSSecurityGroup", {
  vpc,
});
const rdsProxySecurityGroup = new ec2.SecurityGroup(
  this,
  "RDSProxySecurityGroup",
  { vpc },
);
const rdsInstance = new rds.DatabaseInstance(this, "SampleRDS", {
  engine: rds.DatabaseInstanceEngine.MYSQL,
  databaseName: DB_NAME,
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.T3,
    ec2.InstanceSize.SMALL,
  ),
  vpc,
  subnetGroup: new rds.SubnetGroup(this, "RDSSubnetGroup", {
    description: "subnetGroup",
    vpc,
    vpcSubnets: vpc.selectSubnets({
      subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
    }),
  }),
  securityGroups: [rdsSecurityGroup],
  credentials: rds.Credentials.fromSecret(rdsSecret),
});
const rdsProxy = rdsInstance.addProxy("SampleRDSProxy", {
  secrets: [rdsSecret], // RDSのシークレットを指定する
  vpc,
  securityGroups: [rdsProxySecurityGroup],
  requireTLS: false,
});
```

- RDSとRDSProxy用のセキュリティグループを作成します
- RDSProxyのシークレットは新規作成せずに、RDSのシークレットを指定します

## Lambda

```typescript
const lambdaSecurityGroup = new ec2.SecurityGroup(this, "LambdaSecurityGroup", {
  vpc,
});
const libsLayer = new lambda.LayerVersion(this, "SampleLayer", {
  code: lambda.Code.fromAsset("src/libs"),
  compatibleRuntimes: [lambda.Runtime.PYTHON_3_9],
});
const lambdaFunction = new lambda.Function(this, "SampleLambda", {
  runtime: lambda.Runtime.PYTHON_3_9,
  code: lambda.Code.fromAsset("src/functions"),
  handler: "sample.lambda_handler",
  environment: {
    SECRET_ARN: rdsSecret.secretArn,
    PROXY_ENDPOINT: rdsProxy.endpoint, // LambdaはRDSのhostではなくRDSProxyのエンドポイントを参照する
  },
  layers: [libsLayer],
  vpc,
  securityGroups: [lambdaSecurityGroup],
});
rdsSecret.grantRead(lambdaFunction);
rdsInstance.grantConnect(lambdaFunction, DB_USER); // RDSProxyではなくRDSでgrantする
```

- 環境変数でRDSProxyのエンドポイントを渡す
  - secretsManagerでRDSのクレデンシャルを作成すると`host`というキーでhost名も管理できるがRDSProxyを経由する場合はLambdaからは参照する必要なし
- `grant`するのはRDSProxyではなくRDS
  - 個人的にRDSProxyじゃないんかいて思う

## SecutiryGroup

```typescript
rdsProxySecurityGroup.addIngressRule(
  lambdaSecurityGroup,
  ec2.Port.tcp(DB_PORT),
  "Allow from lambda",
);
rdsSecurityGroup.addIngressRule(
  rdsProxySecurityGroup,
  ec2.Port.tcp(DB_PORT),
  "Allow from RDSProxy",
);
```

- RDSProxyセキュリティグループのインバウンドルールでLambdaのセキュリティグループを指定する
- RDSセキュリティグループのインバウンドルールでRDSProxyのセキュリティグループを指定する

## 動作確認してみる

Lambdaのtestから実行

```ログ
[INFO]	2024-06-22T22:12:54.144Z		SUCCESS: Connection to RDS for MySQL instance succeeded
START RequestId: fad5086b-1c70-4e0e-b002-d7deae86a67e Version: $LATEST
[INFO]	2024-06-22T22:12:54.169Z	fad5086b-1c70-4e0e-b002-d7deae86a67e	result=((1, 'custNameSample'))
END RequestId: fad5086b-1c70-4e0e-b002-d7deae86a67e
REPORT RequestId: fad5086b-1c70-4e0e-b002-d7deae86a67e	Duration: 26.82 ms	Billed Duration: 27 ms	Memory Size: 128 MB	Max Memory Used: 72 MB	Init Duration: 518.03 ms
```

RDSへの接続とレコードの追加が確認できました！:tada:

## まとめ

- VPC
  - RDSProxyはサブネットを2つ以上必要とするので2つ以上作成する
  - secretsManagerへのエンドポイントをアタッチする
- RDS
  - RDSインスタンス、RDSProxyそれぞれにセキュリティグループを作成する
  - RDSProxyのシークレットはRDSのシークレットを指定する
- Lambda
  - DB接続時、hostはRDSのhostではなくRDSProxyのエンドポイントを指定する
  - Lambda用のセキュリティグループを作成する
  - secretsManagerで`grantRead()`する
  - RDSインスタンスで`grantConnect()`する
- セキュリティグループ
  - RDSProxyセキュリティグループのインバウンドルールでLambdaのセキュリティグループを指定する
  - RDSセキュリティグループのインバウンドルールでRDSProxyのセキュリティグループを指定する
