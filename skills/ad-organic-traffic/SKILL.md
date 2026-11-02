---
name: ad-organic-traffic
description: >
  Query organic post snapshots and TikTok, Facebook, and Google ad
  performance from kalo_data_website on Metabase (athena_1). Use when the
  user asks for 自然流量, 自然播放, 点赞, 评论, 分享, 收藏, 下载, 广告消耗,
  广告展示, 广告点击, 广告转化, 广告播放, Facebook/Google/TikTok/YouTube
  广告效果, 采集id, or metrics on ods_tiktok_creator_video_detail,
  ods_tiktok_ads_monitor_df, ads_facebook_ads_df, ads_google_ads_df, or
  ods_ad_social_collect_df. Organic counts for platform tiktok and x are
  daily increments between snapshots. Other platforms are historical
  cumulative totals. Ads are usable only if joinable to organic: TikTok
  collect_id = post_id; YouTube collect_id = video_id. When the user groups
  by 内容主题, 主题, or 关联产品,
  read the Feishu JSON: Facebook JSON collect_id joins Facebook ad_id;
  non-Facebook JSON collect_id joins collect_id or aweme_id. When the
  user asks 赛道匹配, 赛道, 赛道分, 内容匹配, 受众重合度, or 档位 for a
  post, read the live JSON in the Feishu doc 赛道分析结果. Before
  installing or querying, confirm the Metabase MCP
  https://bi.kalowave.com/api/metabase-mcp is present; if it is missing,
  ask the user to install it and do not query.
---

# 自然流量与广告流量（Metabase）

用 Metabase MCP 查 `athena_1` / `kalo_data_website`。表号与可加总字段见 [reference.md](reference.md)。

`ads_tiktok_kol_video_di` 的「当天 / 按日 SUM」只属于 `kol-video-daily`。本 skill 的自然流量以 `ods_tiktok_creator_video_detail` 为准。广告表不要按 `partition_day` 相加。

## 安装前检查

安装本 skill，或第一次用它查数之前，先确认已经安装 Metabase MCP。

下面任一情况算已安装：

- MCP 配置里有服务器地址 `https://bi.kalowave.com/api/metabase-mcp`
- 当前会话能看到这个 Metabase MCP。Cursor 里常见名称是 `user-metabase` 或 `metabase`

Cursor 查看 `~/.cursor/mcp.json` 和项目 `.cursor/mcp.json` 的 `mcpServers`。其他 agent 查看它自己的 MCP 配置。地址用上面这一条。

没有这项配置时停下来，不要查表，提示用户安装 Metabase MCP：

https://bi.kalowave.com/api/metabase-mcp

Cursor 的配置写成：

```json
"metabase": {
  "url": "https://bi.kalowave.com/api/metabase-mcp"
}
```

装好后再继续。配置已经存在但未连接时，对这个 MCP 调用一次 `mcp_auth`，再重试。不要连续重复认证。认证仍然失败时，把上面的安装地址再告诉用户。

## 连接

1. `browse_data` → `get_fields` 拿字段数字 id。表号用 reference 里的；对不上就按表名在库 id `10`（`athena_1`）里重查。只用 `schema = kalo_data_website` 且 `db_id = 10` 的那份。
2. 用 `execute_query`（MBQL）出数。表和字段写数字 id，过滤值单独放在子句里。采集 id、帖子 id、广告 id 用字符串，不要转成整数。
3. 两个快照相减、按帖子汇总，用两次查询再算。只有窗口函数或 CTE 才用 `execute_sql`，用户输入放进 `{{tag}}` / `template_tag_values`。`execute_sql` 返回 403 或缺少 `agent:sql:run` 时，告诉用户缺这个权限，改用 `execute_query` 做完。
4. 查数使用已安装的 Metabase MCP（Cursor 里常见名称是 `user-metabase` 或 `metabase`）。未安装时按上一节停下来提示安装，不要直接查表。已安装但未连接时，调用一次 `mcp_auth`，再重试。
5. 用户要图或趋势时，把 `execute_query` 的 `query_handle` 交给 `visualize_query`。图就是结果，不要再查一遍把同一张表贴出来。

`row_limit` 是分页大小。要前 N 行就在 stage 里写 `limit` 并 `order-by`。结果带 `next_cursor` 时继续翻页，不要把一页当成全集。

## 先锁一个快照

`partition_day` 是快照日，不是广告效果日。广告表的一个快照里有多天效果日，相邻快照会重复这些天。

广告表和采集清单先定**一个** `partition_day`。用户没指定时，对该表取最大的 `partition_day`（`yyyyMMdd` 与 `yyyy-MM-dd` 都可以按字符串取最大）。用户指定历史快照时只用那一个。广告跨快照 SUM 会把同一效果日加多次。

自然流量表每个 `partition_day` 是该内容当天的一条记录。`tiktok` / `x` 跨多天时，加的是每天增量，不是各天行上的原值。

各表快照格式：

| 表 | `partition_day` | 效果日 |
|---|---|---|
| `ods_ad_social_collect_df` | `yyyy-MM-dd` | 无。整表是采集清单快照 |
| `ods_tiktok_creator_video_detail` | `yyyyMMdd` | 无单独效果日。`partition_day` 就是自然流量日期。口径按平台，见下节 |
| `ods_tiktok_ads_monitor_df` | `yyyy-MM-dd` | `stat_date`（`yyyy-MM-dd`） |
| `ads_facebook_ads_df` | `yyyyMMdd` | `data_start_date`。当前数据里它等于 `data_end_date` |
| Facebook 设备/国家/版位 | `yyyyMMdd` | `stat_date` |
| Google 各表 | `yyyyMMdd` | `stat_date`（日期；展示成公历日期） |

采集清单、自然流量、广告的最新快照可以不是同一天。回答里分开写。比较日期时把 `yyyyMMdd` 和 `yyyy-MM-dd` 当成同一天，不要按原始字符串硬比。

入库时间（`created_at`、`updated_at`、`gmt_*`、`unixtime_*`）不是投放日期。

## 自然流量

表：`ods_tiktok_creator_video_detail`。同一快照里一个 `aweme_id` 一行。若出现多行，留 `collect_time` 最大的一行。计数字段：`play_count`、`digg_count`、`comment_count`、`share_count`、`collect_count`、`download_count`。

回答时在数字旁标明口径。

`platform` 为 `tiktok` 或 `x`：**每日增量**。行上的原值是截至该快照的水位，当天增量用相邻快照相减，不要把各天的原值 SUM 当成播放量。

- 某一天：该 `partition_day` 减去同一 `aweme_id` 的上一张快照。点赞等可以变少，增量保留负值。写明相减的两个 `partition_day`。
- 一段日期：把窗口内的每日增量相加。缺天时按实际相邻快照逐段相减再相加，缺的天不要补 0。
- 该内容的第一张快照没有上一天，算不出当天增量。这时只报原值，并标明这是首日水位，不是每日增量。

其他 `platform`（含 `youtube`、`facebook`）：**历史累计**。读所问日期那一个快照上的原值。默认最新快照。不要跨 `partition_day` 相加，也不要相减，除非用户明确要增量。

内容不在某个快照里就是没有这条采集。缺快照时说明缺的是哪一天。

`aweme_id` 就是采集 id / 内容 id。

## 广告流量

获取广告数据时，需要限定与自然流量数据能够关联的广告数据，才能被使用。关联关系是，tiktok平台时，collect_id = tiktok广告的post_id，youtube平台时：collect_id等于youtube广告的视频id。

对不上的广告不用，不进合计。先取自然表 `aweme_id`（= `collect_id`）集合，再筛广告。

- TikTok：仅 `ods_tiktok_ads_monitor_df.post_id = collect_id`。`post_id` 为空或对不上自然 `aweme_id` 的行丢弃。
- YouTube：仅 `ads_google_ads_youtube_df.video_id = collect_id`（同样等于自然 `aweme_id`）。`video_id` 为空或对不上的行丢弃。Google 主表只补这些已对上的 `ad_id`，不要拉全量。
- X：没有广告效果表。

在**一个**快照里，按效果日加总可加字段。一个帖子有多条**已关联**广告时，帖子合计是这些广告之和。

粒度：

- TikTok：`ad_id` + `stat_date`。帖子键是 `post_id`。
- Facebook 主表：`ad_id` + `data_start_date`。
- Google 主表：`ad_id` + `stat_date`。

同一广告、同一效果日若有多行，停下来报告重复键和行数，不要直接 SUM。

可加字段与不要加总的比率、均值见 [reference.md](reference.md)。区间点击率、完播率用次数或花费重算。`avg_play_duration` 只在单日行上读。Facebook 的 `reach`、`frequency` 不跨天相加。

币种：

- TikTok 用 `currency`，按币种分开加总。
- Facebook 花费是账户币种，表内没有币种列，不要换算。
- Google 用 `currency_code`，按币种分开加总。`cost` 才是花费。

拆表（设备、国家、版位、关键词、搜索词、展示位置、YouTube、投放网络）只在用户要这个切面时查。不要把拆表花费加进主表花费。

这套表里没有 X 的广告效果表。X 的自然流量仍在创作者视频表。YouTube 广告观看在 `ads_google_ads_youtube_df`，不要把它写成自然播放。Facebook 广告表上的帖子心情、评论是广告侧互动，不要和创作者视频表的评论数相加。

## 采集 id 怎么对

先读采集清单最新快照的一行：`platform`、`creator_account`、`owner_name`、`ops_country`、`tccode`、`post_url`。`collect_id` 按字符串等值匹配。

| 平台 | 自然流量 | 广告（必须先对上自然，才能用） |
|---|---|---|
| `tiktok` | `aweme_id = collect_id` | `ods_tiktok_ads_monitor_df.post_id = collect_id`。空或对不上自然 → 该广告行不用 |
| `x` | `aweme_id = collect_id` | 无广告效果表 |
| `youtube` | 同上，有行才报 | `ads_google_ads_youtube_df.video_id = collect_id`。空或对不上自然 → 该广告行不用；Google 主表只补这些 `ad_id` |
| `facebook` | `aweme_id`，或 `facebook_post_id` / `facebook_creator_post_id` | 采集 id 是帖子 id、`pfbid` 或整段 URL，不是广告 id。用 `user_post_id`（主页 id + `_` + 帖子 id）对帖子。对不上就说明对不上 |

## 按内容主题或关联产品

用户要按内容主题、主题或关联产品汇总时，维度不在 Metabase 里。先读对应 JSON，再用 `collect_id` 回表汇总。不要用 `raw_value`、`resp_data`、`biz_item`，也不要用 `main_topic_tag`、`brand_tags` 或分类字段代替主题和关联产品。

| 范围 | JSON | `collect_id` 对齐 |
|---|---|---|
| Facebook | https://kalowave.feishu.cn/wiki/SSqQwG9dqix7gJkXmooccgnAnSc | `ads_facebook_ads_df.ad_id` |
| 非 Facebook | https://kalowave.feishu.cn/wiki/MKKjwsHSNiZloKkq2FgcXpBKnCf | 采集清单 `collect_id`，或自然流量表 `aweme_id` |

主题取 `core_content.main_topic`。关联产品指的是 `entity_product` 那个字段，路径是 `core_content.entity_product`（字符串数组）。不要改用 `product_id`、`product_keyword` 或其他产品字段。

按 lark-doc 的读取流程打开知识库里的 JSON。只问 Facebook 时只用 Facebook 那份；只问 tiktok、x、youtube 或其他非 Facebook 平台时只用非 Facebook 那份。两边都要时分头取、分头汇总，不要把 Facebook 的 `collect_id` 拿去对 `aweme_id`。

一个内容挂了多个关联产品时，按产品统计会分别落入每个产品。各产品的数不要再加总成全部内容。关联产品为空的单独成组。JSON 里有、表里对不上的 id 单独列出。

对上 id 之后，自然流量和广告仍按前面的口径计算。

## 赛道匹配

用户要查某个帖子的赛道匹配时，数据不在数据库里。每次都重新打开文档「赛道分析结果」，下载里面当时的 JSON 附件再查。不要用上次落在本地的副本，也不要现场重打分。

https://kalowave.feishu.cn/wiki/Fw5yw2embil8yckWS56cx8hzngb

按 lark-doc 打开这篇，下载正文里的 JSON 附件（文件名会变，以文档里当前附件为准）。记录在 `items` 里。用 `内容id` 按字符串等值匹配：

- 采集清单 `ods_ad_social_collect_df.collect_id`
- 自然流量表 `ods_tiktok_creator_video_detail.aweme_id`

一条 `内容id` 一条赛道分析。同一达人账号共用 `赛道`、`赛道分S`、`内容匹配C`。

回报这些字段：`内容id`、`内容url`、`达人账号`、`达人账号主页url`、`平台`、`赛道`、`赛道分S`、`内容匹配C`、`受众重合度`、`档位`、`依据`。空值留空，不要补 0。文档里没有这条 `内容id` 就说明没有，不要改用别的表或 `kol-audience-overlap` 编一个分数。用户明确要求重新打分时才走那个 skill。

`赛道分S`、`内容匹配C`、`受众重合度` 不要相加或平均。

## 回报

写明表名、用的 `partition_day`、效果日窗口和币种。自然流量标明平台口径：`tiktok` / `x` 是每日增量（相邻快照相减，多天再相加），其他平台是历史累计（单个快照的原值）。广告标明是单个快照内按效果日求和。按主题或关联产品汇总时，写明用了哪一份 JSON，以及 `collect_id` 对的是广告 id、采集 id 还是 `aweme_id`。没有行时说明该快照里的日期范围。不要改用另一张表或另一个快照把数补上。
