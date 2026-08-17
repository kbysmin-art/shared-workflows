# shared-workflows

共通CICDリポジトリ。各案件は自分のワークフローで `uses:` からこのリポジトリの
再利用可能ワークフローを呼び出す。クレデンシャル(AWS認証情報)はGitHub Secrets/OIDCとして
各案件側のリポジトリ環境に保持し、このリポジトリ側では受け取ったロールARNでAssumeRoleするだけで、
案件固有の認証情報は持たない。

## ワークフロー一覧

### `.github/workflows/terraform-deploy.yml`

案件のTerraformルートモジュールに対して `init` → `validate` → `plan` →
(`apply` は `apply: true` のときのみ) を実行する。AWSへの認証はGitHub OIDC
(`aws-actions/configure-aws-credentials`)。呼び出し例:

```yaml
jobs:
  deploy:
    uses: kbysmin-art/shared-workflows/.github/workflows/terraform-deploy.yml@main
    with:
      working_directory: terraform/main
      aws_region: ap-northeast-1
      role_to_assume: arn:aws:iam::<account-id>:role/short-scenario-maker-deploy
      backend_config_bucket: short-scenario-maker-tfstate-<account-id>
      backend_config_key: short-scenario-maker/main.tfstate
      backend_config_dynamodb_table: short-scenario-maker-tfstate-lock
      pre_plan_command: npm ci && npm run build:lambda
      tf_vars_json: '{"line_to_id": "${{ vars.LINE_TO_ID }}"}'
      apply: ${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}
    secrets:
      MODULES_GIT_TOKEN: ${{ secrets.MODULES_GIT_TOKEN }}
```

`MODULES_GIT_TOKEN` は `infra-modules` リポジトリが非公開の間だけ必要
(読み取り権限を持つ fine-grained PAT を呼び出し側リポジトリのsecretsに登録する)。
公開リポジトリにする場合は省略可能。

### `.github/workflows/s3-static-deploy.yml`

静的サイト(例: `next build` で `output: "export"` を使うNext.jsアプリ)をビルドし、
S3バケットへ `aws s3 sync --delete` した後、`cloudfront_distribution_id` が
指定されていればキャッシュを invalidate する。AWSへの認証は`terraform-deploy.yml`
と同じくGitHub OIDC。呼び出し例:

```yaml
jobs:
  deploy-site:
    needs: terraform # infra-modules//modules/static-site を適用するジョブの後に実行する
    uses: kbysmin-art/shared-workflows/.github/workflows/s3-static-deploy.yml@main
    with:
      working_directory: src
      build_output_dir: out
      aws_region: ap-northeast-1
      role_to_assume: arn:aws:iam::<account-id>:role/ashitubo-deploy
      s3_bucket: ${{ vars.SITE_BUCKET_NAME }}
      cloudfront_distribution_id: ${{ vars.CLOUDFRONT_DISTRIBUTION_ID }}
      build_env_json: '{"NEXT_PUBLIC_SITE_URL": "${{ vars.NEXT_PUBLIC_SITE_URL }}"}'
```

`s3_bucket` と `cloudfront_distribution_id` は初回 `terraform apply` の出力を
呼び出し側リポジトリの変数(`vars.*`)として登録しておく(`terraform-deploy.yml`の
`backend_config_bucket`などと同じ運用)。このワークフローは`infra-modules`を
参照しないため`MODULES_GIT_TOKEN`は不要。
