# hg38 流程迁移与优化记录

日期：2026-06-22

## 范围

仅修改 `wxs_somatic_gatk_hg38_instant` 目录内文件。`wxs_somatic_gatk_hg19_instant`、`pipeline_optimization_report.md` 和 `需求说明.md` 只作为参考读取。

## 参考依据

- `pipeline_optimization_report.md`
- `wxs_somatic_gatk_hg19_instant/workflow.wdl`
- `wxs_somatic_gatk_hg19_instant/defaults`
- `wxs_somatic_gatk_hg19_instant/inputs`
- `wxs_somatic_gatk_hg19_instant/tasks/*.wdl`
- `wxs_somatic_gatk_hg38_instant/GRCH38_filelist.txt`

## 已修改内容

1. 主流程 `workflow.wdl`
   - 参考 hg19 优化版重组主流程。
   - 新增 `germline_resource` 和 `common_biallelic_variants` 输入。
   - 新增 tumor/normal `GetPileupSummaries`。
   - 新增 `CalculateContamination`。
   - 将 contamination table、tumor segmentation 和 read orientation model 传入 `FilterMutectCalls`。
   - 将三档资源配置替换为 task 级资源配置。

2. 参数模板 `defaults`
   - 保留 hg38 参考路径和文件名。
   - 新增 `germline_resource = af-only-gnomad.hg38.vcf.gz`。
   - 新增 `common_biallelic_variants = small_exac_common_3.hg38.vcf.gz`。
   - 移除未使用的 `bwa_docker_image`。
   - 将 `SMALL/MED/BIG` 三档资源替换为 `cpu4mem8`、`cpu4mem16`、`cpu8mem16`、`cpu16mem32`、`cpu16mem64`、`cpu32mem64`。
   - 新增 Mutect2 分片并行参数。

3. 输入模板 `inputs`
   - 新增 `germline_resource` 和 `common_biallelic_variants`。
   - 新增 task 级资源配置和 Mutect2 并行参数。
   - 移除旧的 `bwa_docker_image` 和三档资源字段。

4. task 文件
   - 迁移 hg19 已优化的 runtime 写法：`instanceTypes` 和 `systemDisk: "cloud " + disk_gb`。
   - 增加任务日志复制：`script.txt`、`stdout.txt`、`stderr.txt`。
   - 多数任务改为在 `/tmp` 本地目录运行，降低 call 目录 I/O 压力。
   - 磁盘估算增加 1000GB 上限保护。
   - `somatic_mutect2.wdl` 增加 germline resource、分 contig 并行、F1R2 输出和 read orientation model。
   - 新增 `get_pileup_summaries.wdl` 和 `calculate_contamination.wdl`。
   - `filter_and_select_pass_variants.wdl` 增加 orientation priors、contamination table、tumor segmentation 和 filtering stats 输出。
   - `annovar_annotation.wdl` 保留 hg38 buildver 和 hg38 注释库协议，并加入空 PASS VCF 兜底输出。

5. 文档
   - 补充 `README.md`，说明 GRCh38 流程、关键资源、WES/WGS 模式和主要输出。
   - 新增本文档记录迁移操作。

6. 仓库清理
   - 确认 `workflow.wdl`、`tasks/*.wdl`、`defaults`、`inputs`、`README.md`、`manifest.json` 和 `schema.json` 未引用 `codescripts/` 与 `pictures/`。
   - `codescripts/` 与 `pictures/` 不属于 hg38 主流程运行所需内容，已从 Git 跟踪中删除。
   - `.gitignore` 保留对 `codescripts/` 与 `pictures/` 的忽略规则，避免后续误提交。

## hg38 资源确认

`GRCH38_filelist.txt` 中已列出本次新增逻辑所需资源：

- `af-only-gnomad.hg38.vcf.gz`
- `af-only-gnomad.hg38.vcf.gz.tbi`
- `small_exac_common_3.hg38.vcf.gz`
- `small_exac_common_3.hg38.vcf.gz.tbi`

未发现当前需求中必须新增但未列出的 hg38 资源。

## 回滚提示

如需回滚本次迁移，可重点恢复以下文件：

- `workflow.wdl`
- `defaults`
- `inputs`
- `tasks/*.wdl`
- `README.md`
- 删除 `MIGRATION_OPTIMIZATION_LOG.md`
- 如需恢复旧脚本或图片，可从删除前提交 `a359e61` 取回 `codescripts/` 与 `pictures/`

如果只想撤销 contamination 相关逻辑，需要同时处理 `workflow.wdl`、`inputs`、`defaults`、`tasks/get_pileup_summaries.wdl`、`tasks/calculate_contamination.wdl` 和 `tasks/filter_and_select_pass_variants.wdl`，否则主流程输入输出会不匹配。
