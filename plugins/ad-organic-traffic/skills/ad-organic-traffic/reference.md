# 表号与字段

库：Metabase `athena_1`（database id `10`），schema `kalo_data_website`。字段 id 每次用 `browse_data` `get_fields` 取，不要手写。

## 表

| 表 | id | 粒度（已先限定一个 `partition_day`） |
|---|---|---|
| `ods_ad_social_collect_df` | 19038 | 一个 `collect_id` 一行。维度表，指标不要加总 |
| `ods_tiktok_creator_video_detail` | 18989 | 一个 `aweme_id` 一行。`tiktok` / `x` 的每日增量 = 相邻快照相减；其他平台是历史累计 |
| `ods_tiktok_ads_monitor_df` | 19072 | `ad_id` × `stat_date`。只用 `post_id = collect_id` 且自然表有该 `aweme_id` 的行 |
| `ads_facebook_ads_df` | 24843 | `ad_id` × `data_start_date` |
| `ads_facebook_ads_device_df` | 24790 | `ad_id` × `stat_date` × `device_platform` |
| `ads_facebook_ads_country_df` | 24780 | `ad_id` × `stat_date` × `country` |
| `ads_facebook_ads_placement_df` | 24810 | `ad_id` × `stat_date` × `publisher_platform` × `placement` |
| `ads_google_ads_df` | 24806 | `ad_id` × `stat_date` |
| `ods_google_ads_keyword_df` | 24807 | 日 × 账户 × 系列 × 广告组 × 关键词 × 匹配类型。无广告 id |
| `ads_google_ads_search_term_df` | 24781 | 再加搜索词。无广告 id。点击为 0 时 `cpc` 为空 |
| `ads_google_ads_placement_df` | 24797 | 日 × 账户 × 系列 × 广告组 × `placement`。无转化列 |
| `ads_google_ads_youtube_df` | 24828 | 日 × 广告 × `video_id`。只用 `video_id = collect_id` 且自然表有该 `aweme_id` 的行 |
| `ads_google_ads_network_df` | 24821 | 再加 `channel_type` × `network`。无转化列 |

关键词、搜索词、展示位置、YouTube 表里，源文件没有的 `headline` / `ad_id` 是空的。Facebook 设备表没有 `device_platform_cn`、`ctr`、`cpm`；国家表和版位表的 `ctr`、`cpm` 以及版位表的 `publisher_platform_cn` 也没有源列。空就留空，需要比率时用花费、展示、点击重算。

## 自然流量

| 字段 | 含义 |
|---|---|
| `play_count` | 播放量 |
| `digg_count` | 点赞量 |
| `comment_count` | 评论量 |
| `share_count` | 分享量 |
| `collect_count` | 收藏量 |
| `download_count` | 下载量 |

`platform` 为 `tiktok` 或 `x` 时，这些字段在每一行上是截至该快照的水位。每日增量 = 本行减去上一快照，多天把增量相加。不要把各天原值 SUM。

其他 `platform` 这些字段是历史累计。读一个 `partition_day` 的原值，不要跨天 SUM。

## 广告必须先关联自然流量

获取广告数据时，需要限定与自然流量数据能够关联的广告数据，才能被使用。关联关系是，tiktok平台时，collect_id = tiktok广告的post_id，youtube平台时：collect_id等于youtube广告的视频id。

对不上自然 `aweme_id` 的广告不 SUM。Google 主表只补已通过 `video_id = collect_id` 对上的 `ad_id`。

## 广告：单个快照内可以 SUM

TikTok `ods_tiktok_ads_monitor_df`：

| 可 SUM | 不要 SUM |
|---|---|
| `spend`、`impression`、`click_cnt`、`convert_cnt` | `avg_play_duration` |
| `video_play_cnt`、`video_play_2s_cnt`、`video_play_6s_cnt`、`focused_play_6s_cnt` | `video_play_rate`、`play_2s_rate`、`play_6s_rate` |
| `video_play_finish_cnt`、`video_play_25_cnt`、`video_play_50_cnt`、`video_play_75_cnt` | `play_25_rate`、`play_50_rate`、`play_75_rate`、`play_finish_rate` |
| `valid_play_cnt`、`play_15s_cnt` | |

Facebook 主表与设备/国家/版位拆表：

| 可 SUM | 不要跨天 SUM |
|---|---|
| `spend`、`impression`、`click_cnt` | `reach`、`frequency` |
| `link_click_cnt`、`post_reaction_cnt`、`post_comment_cnt` | `cpm`、`cpc`、`link_ctr`、`cost_per_link_click`、`ctr` |
| `landing_page_view_cnt`、`add_to_cart_cnt`、`initiate_checkout_cnt` | |
| `purchase_cnt`、`purchase_value`、`complete_registration_cnt` | |
| `start_trial_cnt`、`subscribe_cnt` | |

拆表只有花费、展示、点击。主表上的转化列不要到拆表里找。

Google 主表、关键词、搜索词：

| 可 SUM | 不要 SUM |
|---|---|
| `impressions`、`clicks`、`cost` | `ctr`、`cpc`、`cost_per_conversion`、`roas` |
| `conversions`、`conversion_value` | |

YouTube 额外可 SUM：`video_views`。不要 SUM：`video_view_rate`、`quartile_25_rate`、`quartile_50_rate`、`quartile_75_rate`、`quartile_100_rate`。

展示位置表和投放网络表可 SUM 的是 `impressions`、`clicks`、`cost`，没有转化列。

区间比率示例：完播率 = `SUM(video_play_finish_cnt) / SUM(video_play_cnt)`；点击率 = 点击合计 / 展示合计。分母为 0 就留空。

## 主题与关联产品 JSON

不在库里。按主题或关联产品汇总时整份读取。

| 文件 | 知识库 | `collect_id` |
|---|---|---|
| Facebook | https://kalowave.feishu.cn/wiki/SSqQwG9dqix7gJkXmooccgnAnSc | Facebook 广告 `ad_id` |
| 非 Facebook | https://kalowave.feishu.cn/wiki/MKKjwsHSNiZloKkq2FgcXpBKnCf | 采集清单 `collect_id` 或自然流量 `aweme_id` |

| JSON 路径 | 用途 |
|---|---|
| `collect_id` | 关联键，按字符串等值匹配 |
| `core_content.main_topic` | 内容主题 |
| `core_content.entity_product` | 关联产品。就是 `entity_product` 这个字段，数组 |

## 赛道分析结果 JSON

不在库里。查帖子赛道匹配时，每次重新下载文档里的当前附件。

文档：https://kalowave.feishu.cn/wiki/Fw5yw2embil8yckWS56cx8hzngb

`内容id` 对采集清单 `collect_id`，以及自然流量 `aweme_id`。一条内容 id 一条。

| 字段 | 用途 |
|---|---|
| `内容id` | 关联键 |
| `内容url` | 内容链接 |
| `达人账号` | 达人账号 |
| `达人账号主页url` | 达人主页 |
| `平台` | 平台 |
| `赛道` | 赛道 |
| `赛道分S` | 赛道分，不相加 |
| `内容匹配C` | 内容匹配分，不相加 |
| `受众重合度` | 受众重合度，不相加 |
| `档位` | 匹配档位 |
| `依据` | 打分依据 |
