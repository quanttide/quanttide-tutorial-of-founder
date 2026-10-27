# 在 CLI 中嵌入并安装 Skill

用 agent-skills-rs 实现方案 A
一句话：在编译期用 include_str! 把 SKILL.md 和附属文件交给 register_embedded_skill 注册，运行时构造 InstallConfig 调 install_skill 写入 .agents/skills/<name> 并自动维护 .skill-lock.json。 这是 crate 作者明确设计的能力——canonical 路径、symlink-with-fallback、锁文件、编译期嵌入全在库内完成。
下面按依赖 → 目录结构 → 嵌入注册 → 安装逻辑 → 锁文件 → clap 接入的顺序给出可直接复制的代码，全部基于当前版本 0.3.1 的公开 API。
一、添加依赖
[package]
name = ”mycli“
version = ”1.4.0“
edition = ”2021“
[dependencies]
agent-skills-rs = ”0.3“
clap = { version = ”4“, features = [”derive“] }
anyhow = ”1“
crate 自带 anyhow / clap / serde_yaml / sha2 / walkdir / directories 等依赖，宿主项目不需要重复声明。如果想锁定行为，固定到 0.3.1。
二、项目结构
mycli/
├── Cargo.toml
├── skills/
│   └── deploy-staging/
│       ├── SKILL.md
│       ├── scripts/rollback.sh
│       └── references/checklist.md
└── src/
    ├── main.rs
    └── skills_registry.rs
SKILL.md 必须包含合法 YAML frontmatter，register_embedded_skill 会内部校验 name 和 description，缺任何一个都会返回 Err：
—
name: deploy-staging
description: >
  Deploy the current branch to the staging environment.
  Use when the user asks to deploy or test changes in staging.
—
## Steps
1. Run `mycli deploy —env staging`
2. Verify with `mycli status —env staging`
三、嵌入声明模块
src/skills_registry.rs：
use agent_skills_rs::Skill;
use anyhow::Result;
/// 一个编译期嵌入的 skill
pub struct EmbeddedSkill {
    /// 与 SKILL.md frontmatter 中的 name 一致，作为子命令参数
    pub id: &‘static str,
    pub skill_md: &’static str,
    /// 附属文件：(相对路径, 文本内容)
    pub auxiliary: &‘static [(&’static str, &‘static str)],
}
/// 新增 skill 只需在此登记一条
pub const EMBEDDED: &[EmbeddedSkill] = &[
    EmbeddedSkill {
        id: ”deploy-staging“,
        skill_md: include_str!(”../skills/deploy-staging/SKILL.md“),
        auxiliary: &[
            (”scripts/rollback.sh“, include_str!(”../skills/deploy-staging/scripts/rollback.sh“)),
            (”references/checklist.md“, include_str!(”../skills/deploy-staging/references/checklist.md“)),
        ],
    },
];
/// 把静态声明转换成 crate 的 Skill 类型
pub fn to_skill(entry: &’static EmbeddedSkill) -> Result<Skill> {
    agent_skills_rs::register_embedded_skill(entry.skill_md, entry.auxiliary)
}
pub fn find(id: &str) -> Option<&‘static EmbeddedSkill> {
    EMBEDDED.iter().find(|e| e.id == id)
}
register_embedded_skill 的签名是 (&str, &[(&str, &str)]) -> Result<Skill>，返回的 Skill 携带 name / description / raw_content / auxiliary_files: HashMap<String, String>。
附属文件只支持文本。 auxiliary 是 &str 类型，不能直接 include_bytes! 二进制脚本或图片。如果确实要带二进制，先 base64 编码进源码，运行时在 auxiliary_files 写盘前解码，或者干脆把这类文件放到 GitHub Release 让 skill 正文里指引 Agent 下载。
Windows 注意：include_str! 路径统一用正斜杠相对 crate 根的路径，跨平台都没问题；写入磁盘时也用 / 拼接相对路径。
四、安装逻辑：InstallConfig + install_skill
agent_skills_rs::installer 模块的核心 API：
• InstallConfig::new(canonical_dir: PathBuf) — 构造函数，其他字段取默认值
• InstallConfig 四个字段：mode（Symlink / Copy）、canonical_dir、target_dirs: Vec<PathBuf>、fallback_to_copy: bool
• install_skill(skill: &Skill, config: &InstallConfig) -> Result<InstallResult> — 先把 skill 写到 canonicaldir，再在 targetdirs 里建立链接/复制
• InstallResult { path: PathBuf, symlink_failed: bool }
canonical 目录遵循 crate 约定 .agents/skills/<skill-name>，这是各 Agent 的通用 fallback 路径；target_dirs 用于同时投递到 .claude/skills/、~/.codex/skills/ 等 Agent 首选路径，模式上推荐 Symlink + fallback_to_copy: true，Windows 没有管理员/开发者模式时自动降级为复制。
use agent_skills_rs::{install_skill, InstallConfig, InstallMode, InstallResult};
use std::path::PathBuf;
pub fn install_one(id: &str, project_root: &PathBuf) -> Result<InstallResult> {
    let entry = super::find(id)
        .ok_or_else(|| anyhow::anyhow!(”no embedded skill named `{id}`“))?;
    let skill = super::to_skill(entry)?;
    let canonical = project_root.join(”.agents“).join(”skills“);
    let config = InstallConfig {
        mode: InstallMode::Symlink,
        canonical_dir: canonical.clone(),
        // 同时投递到 Claude Code 首选路径；symlink 失败时自动复制
        target_dirs: vec![project_root.join(”.claude“).join(”skills“)],
        fallback_to_copy: true,
    };
    install_skill(&skill, &config)
}
pub fn install_all(project_root: &PathBuf) -> Result<()> {
    for entry in super::EMBEDDED {
        match install_one(entry.id, project_root) {
            Ok(res) => println!(”✓ {} → {}“, entry.id, res.path.display()),
            Err(e) => eprintln!(”✗ {}: {e}“, entry.id),
        }
    }
    Ok(())
}
只关心 canonical 目录的最简写法：
let config = InstallConfig::new(project_root.join(”.agents/skills“));
agent_skills_rs::install_skill(&skill, &config)?;
五、锁文件：LockManager
install_skill 写完文件后，需要用 LockManager 维护 .agents/.skill-lock.json，它提供 load / save / update_entry / update_entry_with_hash / get_entry / remove_entry 六个方法。锁文件记录每个 skill 的来源、安装路径和确定性 hash，用于升级检测与团队复现。
use agent_skills_rs::{LockManager, Source, SourceType};
use std::path::PathBuf;
pub fn record_install(
    project_root: &PathBuf,
    skill_id: &str,
    skill_path: &PathBuf,
    cli_version: &str,
) -> anyhow::Result<()> {
    let lock_path = project_root.join(”.agents“).join(”.skill-lock.json“);
    let mgr = LockManager::new(lock_path);
    let source = Source {
        source_type: SourceType::Embedded,
        // 其余字段按 Source 结构体实际定义填充，
        // 这里用 crate 版本作为占位标识
        ..Default::default()
    };
    mgr.update_entry(skill_id, &source, skill_path)?;
    let _ = cli_version;
    Ok(())
}
Source / SourceType 的完整字段名以 docs.rs 为准（crate 文档覆盖度约 38%，个别类型需要直接看源码）。如果想跳过锁文件，可以只在 CLI 启动时打印”已嵌入 skill vX.Y.Z“的提示，但要付出升级后旧目录静默不一致的代价。
六、clap 子命令接入
src/main.rs：
mod skills_registry;
mod install;
use clap::{Parser, Subcommand};
#[derive(Parser)]
#[command(version, about)]
struct Cli {
    #[command(subcommand)]
    cmd: Cmd,
}
#[derive(Subcommand)]
enum Cmd {
    /// Manage embedded agent skills
    Skill {
        #[command(subcommand)]
        action: SkillAction,
    },
}
#[derive(Subcommand)]
enum SkillAction {
    /// Install one or all embedded skills to ./.agents/skills
    Install {
        /// Skill id; omit to install all
        id: Option<String>,
        /// Also install to ~/.agents/skills for cross-project use
        #[arg(long)]
        global: bool,
    },
    /// List embedded skills and their descriptions
    List,
}
fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();
    let project_root = std::env::current_dir()?;
    match cli.cmd {
        Cmd::Skill { action } => match action {
            SkillAction::Install { id, global } => {
                let root = if global {
                    dirs::home_dir().expect(”no home dir“)
                } else {
                    project_root
                };
                match id {
                    Some(id) => {
                        let r = install::install_one(&id, &root)?;
                        println!(”✓ installed {id} → {}“, r.path.display());
                        if r.symlink_failed {
                            println!(”  (symlink unavailable, fell back to copy)“);
                        }
                    }
                    None => install::install_all(&root)?,
                }
            }
            SkillAction::List => {
                for entry in skills_registry::EMBEDDED {
                    let skill = skills_registry::to_skill(entry)?;
                    println!(”{}  v{}  {}“, entry.id, env!(”CARGO_PKG_VERSION“), skill.description);
                }
            }
        },
    }
    Ok(())
}
使用效果：
$ mycli skill list
deploy-staging  v1.4.0  Deploy the current branch to the staging environment...
$ mycli skill install
✓ deploy-staging → ./.agents/skills/deploy-staging
$ mycli skill install deploy-staging —global
✓ deploy-staging → /home/you/.agents/skills/deploy-staging
七、升级检测
get_embedded_skills / get_embedded_skill 可以在运行时取出所有已注册 skill。配合锁文件可以在 CLI 启动时检测”项目里已安装的 skill hash 与当前二进制嵌入内容是否一致“，不一致则提示重装：
pub fn check_stale(project_root: &PathBuf) -> anyhow::Result<()> {
    let lock_path = project_root.join(”.agents“).join(”.skill-lock.json“);
    let mgr = LockManager::new(lock_path);
    for entry in skills_registry::EMBEDDED {
        if mgr.get_entry(entry.id)?.is_some() {
            let on_disk = project_root
                .join(”.agents/skills“).join(entry.id).join(”SKILL.md“);
            if let Ok(existing) = std::fs::read_to_string(&on_disk) {
                if existing != entry.skill_md {
                    eprintln!(
                        ”note: skill `{}` differs from embedded v{}; \
                         run `mycli skill install {} —force`“,
                        entry.id, env!(”CARGO_PKG_VERSION“), entry.id
                    );
                }
            }
        }
    }
    Ok(())
}
把 check_stale 挂在 mycli —version 或主命令入口处，代价极低。—force 的语义就是在 install_skill 之前先 LockManager::remove_entry 并清空目录。
八、CI / 发布注意点
1. 测试中校验注册结果，防止改名或漏 frontmatter 在运行时才爆：
#[test]
fn all_embedded_skills_parse() {
    for entry in skills_registry::EMBEDDED {
        let s = skills_registry::to_skill(entry)
            .unwrap_or_else(|e| panic!(”skill `{}` invalid: {e}“, entry.id));
        assert_eq!(s.name, entry.id);
        assert!(s.description.len() > 10);
    }
}
2. cargo publish 前用同作者的 agent-skills crate 跑一次独立校验——agent_skills::Skill::parse(content) 可以在 CI 里作为 lint 阶段跑。
3. 二进制分发零额外步骤。skill 已在 binary 里，cargo-dist / GitHub Release / brew / cargo install 不需要任何改动，这是方案 A 最大的红利。
4. README 引导一行：
mycli skill install   # make SKILL.md discoverable to Claude Code / Codex / Cursor
这套实现的全部业务代码集中在 skills_registry.rs + 一个安装模块，新增 skill 只需复制目录、登记一个 EmbeddedSkill 常量。