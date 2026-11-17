#pragma newdecls required
#pragma semicolon 1

#include <sourcemod>
#include <sdktools>
#include <left4dhooks>

#define PLUGIN_VERSION "2.3.5-360test"

#define TEAM_SURVIVORS 2
#define TEAM_INFECTED  3

#define GAMEMODE_COOP     1
#define GAMEMODE_VERSUS   2
#define GAMEMODE_SURVIVAL 3

#define ZC_SMOKER  1
#define ZC_BOOMER  2
#define ZC_HUNTER  3
#define ZC_SPITTER 4
#define ZC_JOCKEY  5
#define ZC_CHARGER 6

#define IDLE_CHECK_INTERVAL  0.5
#define IDLE_SPEED_THRESHOLD 5.0
#define TELEPORT_DISTANCE_THRESHOLD 300.0

#define SPAWN_DELAY_CAP_HARD 120.0
#define SPAWN_DELAY_MIN      0.1

#define SPAWN_SEP_MAX_POINTS     16
#define SPAWN_FALLBACK_SAFE_DIST 350.0
#define SPAWN_FIRSTWAVE_TOPUP    1.0

// ===== 找位：恢复 2.3.0 的 360° 三阶段扫描 =====
#define SCAN_ANGLE_STEP      20.0       // 18 个方向
#define SCAN_NEAR_RINGS      3          // 原近距 3 环
#define SCAN_FAR_RINGS       2          // 原远距 2 环
#define SCAN_MAX_CAND        64         // 原近距候选缓存上限
#define SCAN_MAX_FAR_CAND    40
#define SCAN_MAX_PICK        6

#define SCAN_CHEAP_MS        5
#define SCAN_VERIFY_MS       6
#define SCAN_VIS_MS          4
#define SCAN_FAR_MS          5
#define SCAN_GLOBAL_MS       18

#define SCAN_PATH_NEAR       6
#define SCAN_PATH_VIS        4
#define SCAN_PATH_FAR        4

#define SCAN_MIN_TRACES      6
#define SCAN_MAX_TRACES      80
#define SCAN_FAR_MAX_TRACES  60
#define SCAN_FAR_EXT         600.0

// 仅给②可视兜底额外采样；①不可视刷特的最大距离仍然是 spawn_max_range。
#define VIS_SCAN_EXTEND             300.0
#define VIS_SCAN_DIRS               12
#define VIS_SCAN_MAX_POINTS         12
#define VIS_SCAN_EXTRA_MS           1
#define VIS_SCAN_EXTRA_MAX_TRACES   36

#define SPAWN_MAX_TRACE_TIME_MS 6

#define HEAVY_SEARCH_GAP     0.30

#define CLASS_COOLDOWN       1.5
#define CLASS_RESERVE_TTL    3.0

#define RETRY_BASE           1.5
#define RETRY_STEP           1.0
#define RETRY_MAX            4.0

#define RAY_DOWN_ANGLE view_as<float>({90.0, 0.0, 0.0})
#define NAV_HEIGHT    20.0
#define PLAYER_HEIGHT 72.0

#define IB_NAV_SPAWN_FINALE         (1 << 6)
#define IB_NAV_SPAWN_CHECKPOINT     (1 << 11)
#define IB_NAV_SPAWN_RESCUE_VEHICLE (1 << 15)
#define IB_NAV_SPAWN_RESCUE_CLOSET  (1 << 16)

#if !defined FLOAT_PI
#define FLOAT_PI 3.14159265358979323846
#endif

enum
{
    SECTOR_FRONT,
    SECTOR_REAR,
    SECTOR_RIGHT,
    SECTOR_LEFT
};

public Plugin myinfo =
{
    name        = "[L4D/L4D2] Infected Bots Control",
    author      = "djromero (SkyDavid), MI 5, ivanlee（汉化）, 逻辑修复版",
    description = "无限特感修复版 + 旧版360扫描 + 扩展可视兜底 + 防重复同类",
    version     = PLUGIN_VERSION,
    url         = "http://il4d2.lofter.com/"
};

// ===== Globals =====
int  g_iInfectedRealCount, g_iInfectedBotCount, g_iInfectedBotQueue;
int  g_iGameMode, g_iBoomerLimit, g_iSmokerLimit, g_iHunterLimit;
int  g_iSpitterLimit, g_iJockeyLimit, g_iChargerLimit;
int  g_iDefaultMaxPlayerZombies, g_iMaxPlayerZombies, g_iBotReady;
int  g_iZombieClassTank, g_iPlayersInServer;
float g_fInfectedSpawnTimeMax, g_fInfectedSpawnTimeMin;
int  g_iInitialSpawnInt;
int  g_iTankLimit, g_iOnlyClient, g_iLastSurvivors = -1, g_iCurrentIncapCount = 0;

bool g_bHasRoundStarted, g_bHasRoundEnded, g_bLeftSaveRoom;
bool g_bCanSpawnBoomer, g_bCanSpawnSmoker, g_bCanSpawnHunter;
bool g_bCanSpawnSpitter, g_bCanSpawnJockey, g_bCanSpawnCharger;
bool g_bDirectorSpawn, g_bSpecialHalt, g_bInitialSpawn, g_bIsL4D2;
bool g_bAlreadyGhosted[MAXPLAYERS + 1], g_bAlreadyGhostedBot[MAXPLAYERS + 1];
bool g_bDirectorCvarsModified, g_bPlayerHasEnteredStart[MAXPLAYERS + 1];
bool g_bAdjustSpawnTimes, g_bCoordination, g_bDisableSpawnsTank, g_bOldData;
bool g_bIsIncap[MAXPLAYERS + 1];
bool g_bTankBonusActive = false;
bool g_bLastSpawnIsFallback = false;
bool g_bLateLoad = false;

bool g_bFirstWaveArmed  = false;
bool g_bFirstWaveLogged = false;

int   g_iClassReserved[9];
float g_fReserveExpire[9];
float g_fClassCooldown[9];
int   g_iSpawnFailStreak = 0;

float g_fSpawnLifeTime[MAXPLAYERS + 1];
bool  g_bPendingLifespanKill[MAXPLAYERS + 1];
float g_fBotFirstSeen[MAXPLAYERS + 1];
float g_fGhostSince[MAXPLAYERS + 1];
float g_fLastSpawnSuccess  = 0.0;
float g_fLastWatchdogReset = 0.0;

Handle g_hSpawnTimer[MAXPLAYERS + 1];
bool   g_bSpawnReady[MAXPLAYERS + 1];
float  g_fSpawnTimerDeadline[MAXPLAYERS + 1];

Handle g_hDebouncePrintTimer = null, g_hPeriodicCheckTimer = null;
Handle g_hIdleCheckTimer = null;

float g_fIdleTime[MAXPLAYERS + 1], g_fLastPos[MAXPLAYERS + 1][3];
float g_fSpawnSepPos[SPAWN_SEP_MAX_POINTS][3];
float g_fSpawnSepTime[SPAWN_SEP_MAX_POINTS];
int   g_iSpawnSepCount = 0;
int   g_iSpawnSepHead  = 0;

ArrayList g_hHurtTriggers = null;
float     g_fHurtRefreshTime = 0.0;

StringMap g_hSavedCvars = null;
int       g_iSavedZMaxFlags = -1;
bool      g_bSavedZMaxHadUpper = false;
float     g_fSavedZMaxUpper = 0.0;

static const float g_fDownOffsets[] = { 0.0, -90.0, 100.0 };

// ===== ConVars =====
ConVar g_hCvarBoomerLimit, g_hCvarSmokerLimit, g_hCvarHunterLimit;
ConVar g_hCvarSpitterLimit, g_hCvarJockeyLimit, g_hCvarChargerLimit;
ConVar g_hCvarPluginMaxSpecials, g_hCvarInfectedSpawnTimeMax;
ConVar g_hCvarInfectedSpawnTimeMin, g_hCvarDirectorSpawn, g_hCvarGameMode;
ConVar g_hCvarCoordination, g_hCvarIdletimeB4Slay, g_hCvarInitialSpawn;
ConVar g_hCvarBotGhostTime, g_hCvarDisableSpawnsTank, g_hCvarTankLimit;
ConVar g_hCvarAdjustSpawnTimes, g_hZMaxPlayerZombies;
ConVar g_hCvarSpawnMinRange, g_hCvarSpawnMaxRange, g_hCvarIncapSpawnDelay, g_hCvarTankSpawnDelay;
ConVar g_hCvarSpawnSepRadius, g_hCvarSpawnSepTTL, g_hCvarSpawnMaxScanDist;
ConVar g_hCvarSpawnVisMinRange;
ConVar g_hCvarAIEnabled, g_hCvarAIDefaultPlayerCount;
ConVar g_hCvarAIPlayerCountOfAutoAdd, g_hCvarAICountOfAutoAdd;
ConVar g_hCvarMaxLifetime, g_hCvarLifespanGrace;

char g_sSpawnCmd[32];

// ===== Load =====
public APLRes AskPluginLoad2(Handle myself, bool late, char[] error, int err_max)
{
    char sGameName[64];
    GetGameFolderName(sGameName, sizeof(sGameName));

    if (StrContains(sGameName, "left4dead", false) == -1)
    {
        strcopy(error, err_max, "Plugin only supports Left 4 Dead 1 & 2.");
        return APLRes_Failure;
    }

    if (StrEqual(sGameName, "left4dead2", false)) g_bIsL4D2 = true;

    g_bLateLoad = late;

    RegPluginLibrary("infectedbots_fix");
    CreateNative("IB_NotifyInfectedTeleported", Native_NotifyInfectedTeleported);
    return APLRes_Success;
}

public int Native_NotifyInfectedTeleported(Handle plugin, int numParams)
{
    int client = GetNativeCell(1);
    if (client < 1 || client > MaxClients) return 0;
    if (!IsClientInGame(client)) return 0;
    if (GetClientTeam(client) != TEAM_INFECTED) return 0;

    g_fSpawnLifeTime[client] = GetGameTime();
    g_bPendingLifespanKill[client] = false;
    return 1;
}

public void OnPluginStart()
{
    g_iZombieClassTank = g_bIsL4D2 ? 8 : 5;
    strcopy(g_sSpawnCmd, sizeof(g_sSpawnCmd), g_bIsL4D2 ? "z_spawn_old" : "z_spawn");

    CreateConVar("l4d_infectedbots_version", PLUGIN_VERSION, "Version",
        FCVAR_SPONLY | FCVAR_REPLICATED | FCVAR_NOTIFY | FCVAR_DONTRECORD);

    g_hCvarGameMode      = FindConVar("mp_gamemode");
    g_hZMaxPlayerZombies = FindConVar("z_max_player_zombies");
    if (g_hCvarGameMode == null)      SetFailState("Failed to find convar: mp_gamemode");
    if (g_hZMaxPlayerZombies == null) SetFailState("Failed to find convar: z_max_player_zombies");

    g_hCvarBoomerLimit = CreateConVar("l4d_infectedbots_boomer_limit", "5", "Boomer数量", FCVAR_NOTIFY);
    g_hCvarSmokerLimit = CreateConVar("l4d_infectedbots_smoker_limit", "5", "Smoker数量", FCVAR_NOTIFY);
    g_hCvarTankLimit   = CreateConVar("l4d_infectedbots_tank_limit",   "0", "Tank数量", FCVAR_NOTIFY);

    if (g_bIsL4D2)
    {
        g_hCvarSpitterLimit = CreateConVar("l4d_infectedbots_spitter_limit", "5", "Spitter数量", FCVAR_NOTIFY);
        g_hCvarJockeyLimit  = CreateConVar("l4d_infectedbots_jockey_limit",  "5", "Jockey数量", FCVAR_NOTIFY);
        g_hCvarChargerLimit = CreateConVar("l4d_infectedbots_charger_limit", "5", "Charger数量", FCVAR_NOTIFY);
    }
    g_hCvarHunterLimit = CreateConVar("l4d_infectedbots_hunter_limit", "5", "Hunter数量", FCVAR_NOTIFY);

    g_hCvarPluginMaxSpecials = CreateConVar("l4d_infectedbots_max_specials", "3", "特感总数量", FCVAR_NOTIFY);

    g_hCvarInfectedSpawnTimeMax = CreateConVar("l4d_infectedbots_spawn_time_max", "8.0",
        "最大刷新间隔（秒，支持小数，最小 0.1）", FCVAR_NOTIFY, true, SPAWN_DELAY_MIN);
    g_hCvarInfectedSpawnTimeMin = CreateConVar("l4d_infectedbots_spawn_time_min", "6.0",
        "最小刷新间隔（秒，支持小数，最小 0.1）", FCVAR_NOTIFY, true, SPAWN_DELAY_MIN);

    g_hCvarDirectorSpawn = CreateConVar("l4d_infectedbots_director_spawn_times", "0",
        "0=插件刷特, 1=导演刷特", FCVAR_NOTIFY, true, 0.0, true, 1.0);
    g_hCvarCoordination = CreateConVar("l4d_infectedbots_coordination", "0", "协调模式",
        FCVAR_NOTIFY, true, 0.0, true, 1.0);

    g_hCvarIdletimeB4Slay = CreateConVar("l4d_infectedbots_lifespan", "10",
        "静止多少秒处死(0=禁用)", FCVAR_NOTIFY, true, 0.0);
    g_hCvarMaxLifetime = CreateConVar("l4d_infectedbots_max_lifetime", "30",
        "特感存活多少秒后强制处死(0=禁用)，传送会重置计时，控制幸存者时暂停", FCVAR_NOTIFY, true, 0.0);
    g_hCvarLifespanGrace = CreateConVar("l4d_infectedbots_lifespan_grace", "8",
        "新刷特感多少秒内豁免处死(0=禁用)", FCVAR_NOTIFY, true, 0.0);

    g_hCvarInitialSpawn = CreateConVar("l4d_infectedbots_initial_spawn_timer", "10", "首发延迟", FCVAR_NOTIFY);
    g_hCvarBotGhostTime = CreateConVar("l4d_infectedbots_ghost_time", "2", "对抗幽灵时间", FCVAR_NOTIFY);

    g_hCvarDisableSpawnsTank = CreateConVar("l4d_infectedbots_spawns_disabled_tank", "0",
        "坦克在场停止刷特", FCVAR_NOTIFY, true, 0.0, true, 1.0);
    g_hCvarAdjustSpawnTimes = CreateConVar("l4d_infectedbots_adjust_spawn_times", "1",
        "按人数自动调整刷新间隔", FCVAR_NOTIFY, true, 0.0, true, 1.0);

    g_hCvarSpawnMinRange = CreateConVar("l4d_infectedbots_spawn_min_range", "435.0",
        "生成最小距离", FCVAR_NOTIFY, true, 0.0, true, 5000.0);
    g_hCvarSpawnMaxRange = CreateConVar("l4d_infectedbots_spawn_max_range", "1250.0",
        "生成最大距离（0=用 min_range+1000）", FCVAR_NOTIFY, true, 0.0, true, 10000.0);

    // [v2.3.4] 可视刷特最小距离
    g_hCvarSpawnVisMinRange = CreateConVar("l4d_infectedbots_spawn_vis_min_range", "1250.0",
        "可视兜底最小距离；自动不会低于 spawn_max_range，默认1250",
        FCVAR_NOTIFY, true, 0.0, true, 5000.0);

    g_hCvarSpawnSepRadius = CreateConVar("l4d_infectedbots_spawn_sep_radius", "150.0",
        "最近刷点分散半径（0=禁用）", FCVAR_NOTIFY, true, 0.0, true, 500.0);
    g_hCvarSpawnSepTTL = CreateConVar("l4d_infectedbots_spawn_sep_ttl", "1.5",
        "最近刷点分散记忆秒数", FCVAR_NOTIFY, true, 0.0, true, 10.0);
    g_hCvarSpawnMaxScanDist = CreateConVar("l4d_infectedbots_spawn_max_scan_dist", "1850.0",
        "单次扫描最大距离", FCVAR_NOTIFY, true, 500.0, true, 5000.0);

    g_hCvarIncapSpawnDelay = CreateConVar("l4d_infectedbots_incap_spawn_delay", "1.0",
        "倒地加成秒数(0=禁用)", FCVAR_NOTIFY, true, 0.0);
    g_hCvarTankSpawnDelay = CreateConVar("l4d_infectedbots_tank_spawn_delay", "1.0",
        "坦克存活时加成秒数(0=禁用)", FCVAR_NOTIFY, true, 0.0);

    g_hCvarAIEnabled = CreateConVar("l4d_ai_enabled_infectedbots", "1", "启用智能模式",
        FCVAR_NOTIFY, true, 0.0, true, 1.0);
    g_hCvarAIDefaultPlayerCount   = CreateConVar("l4d_ai_DefPlayerCount_infectedbots", "3", "智能模式基准人数", FCVAR_NOTIFY);
    g_hCvarAIPlayerCountOfAutoAdd = CreateConVar("l4d_ai_playercountOfAutoAdd_infectedbots", "1", "每人加组数", FCVAR_NOTIFY);
    g_hCvarAICountOfAutoAdd       = CreateConVar("l4d_ai_conuntOfAutoAdd_infectedbots", "1", "每组加特数", FCVAR_NOTIFY);

    g_iBoomerLimit = g_hCvarBoomerLimit.IntValue;
    g_iSmokerLimit = g_hCvarSmokerLimit.IntValue;
    g_iHunterLimit = g_hCvarHunterLimit.IntValue;
    if (g_bIsL4D2)
    {
        g_iSpitterLimit = g_hCvarSpitterLimit.IntValue;
        g_iJockeyLimit  = g_hCvarJockeyLimit.IntValue;
        g_iChargerLimit = g_hCvarChargerLimit.IntValue;
    }

    g_iDefaultMaxPlayerZombies = g_hCvarPluginMaxSpecials.IntValue;
    g_iMaxPlayerZombies        = g_hCvarPluginMaxSpecials.IntValue;
    g_bDirectorSpawn           = g_hCvarDirectorSpawn.BoolValue;
    g_bAdjustSpawnTimes        = g_hCvarAdjustSpawnTimes.BoolValue;
    g_bCoordination            = g_hCvarCoordination.BoolValue;
    g_bDisableSpawnsTank       = g_hCvarDisableSpawnsTank.BoolValue;
    g_fInfectedSpawnTimeMax    = g_hCvarInfectedSpawnTimeMax.FloatValue;
    g_fInfectedSpawnTimeMin    = g_hCvarInfectedSpawnTimeMin.FloatValue;
    g_iInitialSpawnInt         = g_hCvarInitialSpawn.IntValue;
    g_iTankLimit               = g_hCvarTankLimit.IntValue;

    g_hCvarBoomerLimit.AddChangeHook(ConVarBoomerLimit);
    g_hCvarSmokerLimit.AddChangeHook(ConVarSmokerLimit);
    g_hCvarHunterLimit.AddChangeHook(ConVarHunterLimit);
    if (g_bIsL4D2)
    {
        g_hCvarSpitterLimit.AddChangeHook(ConVarSpitterLimit);
        g_hCvarJockeyLimit.AddChangeHook(ConVarJockeyLimit);
        g_hCvarChargerLimit.AddChangeHook(ConVarChargerLimit);
    }
    g_hCvarPluginMaxSpecials.AddChangeHook(ConVarMaxPlayerZombies);
    g_hCvarDirectorSpawn.AddChangeHook(ConVarDirectorSpawn);
    g_hCvarGameMode.AddChangeHook(ConVarGameMode);
    g_hCvarAdjustSpawnTimes.AddChangeHook(ConVarAdjustSpawnTimes);
    g_hCvarCoordination.AddChangeHook(ConVarCoordination);
    g_hCvarDisableSpawnsTank.AddChangeHook(ConVarDisableSpawnsTank);
    g_hCvarInfectedSpawnTimeMax.AddChangeHook(ConVarInfectedSpawnTimeMax);
    g_hCvarInfectedSpawnTimeMin.AddChangeHook(ConVarInfectedSpawnTimeMin);
    g_hCvarInitialSpawn.AddChangeHook(ConVarInitialSpawn);
    g_hCvarTankLimit.AddChangeHook(ConVarTankLimit);

    HookDirectorCvar("z_hunter_limit");
    if (g_bIsL4D2)
    {
        HookDirectorCvar("z_smoker_limit");
        HookDirectorCvar("z_boomer_limit");
        HookDirectorCvar("z_jockey_limit");
        HookDirectorCvar("z_spitter_limit");
        HookDirectorCvar("z_charger_limit");
        HookDirectorCvar("survival_max_boomers");
        HookDirectorCvar("survival_max_smokers");
        HookDirectorCvar("survival_max_hunters");
        HookDirectorCvar("survival_max_jockeys");
        HookDirectorCvar("survival_max_spitters");
        HookDirectorCvar("survival_max_chargers");
        HookDirectorCvar("survival_max_specials");
    }
    else
    {
        HookDirectorCvar("z_gas_limit");
        HookDirectorCvar("z_exploding_limit");
        HookDirectorCvar("holdout_max_boomers");
        HookDirectorCvar("holdout_max_smokers");
        HookDirectorCvar("holdout_max_hunters");
        HookDirectorCvar("holdout_max_specials");
    }

    HookEvent("round_start",               evtRoundStart, EventHookMode_PostNoCopy);
    HookEvent("round_end",                 evtRoundEnd,   EventHookMode_Pre);
    HookEvent("player_death",              evtPlayerDeath, EventHookMode_Pre);
    HookEvent("player_team",               evtPlayerTeam);
    HookEvent("player_spawn",              evtPlayerSpawn);
    HookEvent("revive_success",            evtReviveSuccess);
    HookEvent("player_incapacitated",      evtPlayerIncapacitated);
    HookEvent("create_panic_event",        evtSurvivalStart);
    HookEvent("finale_start",              evtFinaleStart);
    HookEvent("player_bot_replace",        evtBotReplacedPlayer);
    HookEvent("player_first_spawn",        evtPlayerFirstSpawned);
    HookEvent("player_entered_start_area", evtPlayerFirstSpawned);
    HookEvent("player_entered_checkpoint", evtPlayerFirstSpawned);
    HookEvent("player_transitioned",       evtPlayerFirstSpawned);
    HookEvent("player_left_start_area",    evtPlayerFirstSpawned);
    HookEvent("player_left_checkpoint",    evtPlayerFirstSpawned);
    HookEvent("tank_spawn",                evtTankSpawn);
    HookEvent("tank_killed",               evtTankKilled);

    RegAdminCmd("sm_ibdebug", Cmd_IBDebug, ADMFLAG_ROOT, "IB 诊断");
    RegAdminCmd("sm_ibreset", Cmd_IBReset, ADMFLAG_ROOT, "IB 强制重置调度池");

    AutoExecConfig(true, "l4d2_infectedbots_fix_ch", "sourcemod");
}

void HookDirectorCvar(const char[] name)
{
    ConVar cv = FindConVar(name);
    if (cv != null) cv.AddChangeHook(ConVarDirectorCvarChanged);
}

void EnableAllSpawnFlags()
{
    g_bCanSpawnBoomer = true;
    g_bCanSpawnSmoker = true;
    g_bCanSpawnHunter = true;
    if (g_bIsL4D2)
    {
        g_bCanSpawnSpitter = true;
        g_bCanSpawnJockey  = true;
        g_bCanSpawnCharger = true;
    }
}

public void OnAllPluginsLoaded()
{
    if (g_bLateLoad) RequestFrame(Frame_LateInit);
}

void Frame_LateInit()
{
    GameModeCheck();

    g_iPlayersInServer = 0;
    for (int i = 1; i <= MaxClients; i++)
        if (IsClientInGame(i) && !IsFakeClient(i)) g_iPlayersInServer++;

    if (TrueNumberOfSurvivors() < 1) return;

    g_bHasRoundEnded     = false;
    g_bHasRoundStarted   = true;
    g_fLastSpawnSuccess  = GetGameTime();
    g_fLastWatchdogReset = 0.0;

    if (!g_bDirectorSpawn) TweakSettings(); else DirectorStuff();

    Timer_PlayerLeftStart(null);

    LogMessage("[IB] v%s late load initialized (players=%d)", PLUGIN_VERSION, g_iPlayersInServer);
}

public void OnPluginEnd()
{
    RestoreAllSavedCvars();

    if (g_hZMaxPlayerZombies != null)
    {
        if (g_iSavedZMaxFlags != -1)
            g_hZMaxPlayerZombies.Flags = g_iSavedZMaxFlags;
        if (g_bSavedZMaxHadUpper)
            g_hZMaxPlayerZombies.SetBounds(ConVarBound_Upper, true, g_fSavedZMaxUpper);
    }

    if (g_hSavedCvars != null) { delete g_hSavedCvars; g_hSavedCvars = null; }

    if (g_hPeriodicCheckTimer != null) { delete g_hPeriodicCheckTimer; g_hPeriodicCheckTimer = null; }
    if (g_hIdleCheckTimer != null)     { delete g_hIdleCheckTimer;     g_hIdleCheckTimer = null; }
    if (g_hDebouncePrintTimer != null) { delete g_hDebouncePrintTimer; g_hDebouncePrintTimer = null; }
    if (g_hHurtTriggers != null)       { delete g_hHurtTriggers;       g_hHurtTriggers = null; }

    ClearAllSpawnTimers();
}

// ===== cvar 快照 / 还原 =====
void SaveCvarOnce(const char[] name)
{
    if (g_hSavedCvars == null) g_hSavedCvars = new StringMap();
    float dummy;
    if (g_hSavedCvars.GetValue(name, dummy)) return;

    ConVar cv = FindConVar(name);
    if (cv == null) return;
    g_hSavedCvars.SetValue(name, cv.FloatValue);
}

void SetCvarInt(const char[] name, int value)
{
    ConVar cv = FindConVar(name);
    if (cv == null) return;
    SaveCvarOnce(name);
    cv.SetInt(value);
}

void SetCvarFloat(const char[] name, float value)
{
    ConVar cv = FindConVar(name);
    if (cv == null) return;
    SaveCvarOnce(name);
    cv.SetFloat(value);
}

void RestoreDefaultSafe(const char[] name)
{
    ConVar cv = FindConVar(name);
    if (cv != null) cv.RestoreDefault();
}

void RestoreAllSavedCvars()
{
    if (g_hSavedCvars == null) return;

    StringMapSnapshot snap = g_hSavedCvars.Snapshot();
    for (int i = 0; i < snap.Length; i++)
    {
        int len = snap.KeyBufferSize(i);
        char[] key = new char[len];
        snap.GetKey(i, key, len);

        float v;
        if (g_hSavedCvars.GetValue(key, v))
        {
            ConVar cv = FindConVar(key);
            if (cv != null) cv.SetFloat(v);
        }
    }
    delete snap;
}

// ===== Cvar hooks =====
public void ConVarBoomerLimit(ConVar c, const char[] o, const char[] n)  { g_iBoomerLimit = g_hCvarBoomerLimit.IntValue; }
public void ConVarSmokerLimit(ConVar c, const char[] o, const char[] n)  { g_iSmokerLimit = g_hCvarSmokerLimit.IntValue; }
public void ConVarHunterLimit(ConVar c, const char[] o, const char[] n)  { g_iHunterLimit = g_hCvarHunterLimit.IntValue; }
public void ConVarSpitterLimit(ConVar c, const char[] o, const char[] n) { g_iSpitterLimit = g_hCvarSpitterLimit.IntValue; }
public void ConVarJockeyLimit(ConVar c, const char[] o, const char[] n)  { g_iJockeyLimit = g_hCvarJockeyLimit.IntValue; }
public void ConVarChargerLimit(ConVar c, const char[] o, const char[] n) { g_iChargerLimit = g_hCvarChargerLimit.IntValue; }
public void ConVarTankLimit(ConVar c, const char[] o, const char[] n)    { g_iTankLimit = g_hCvarTankLimit.IntValue; }
public void ConVarInfectedSpawnTimeMax(ConVar c, const char[] o, const char[] n) { g_fInfectedSpawnTimeMax = g_hCvarInfectedSpawnTimeMax.FloatValue; }
public void ConVarInfectedSpawnTimeMin(ConVar c, const char[] o, const char[] n) { g_fInfectedSpawnTimeMin = g_hCvarInfectedSpawnTimeMin.FloatValue; }
public void ConVarInitialSpawn(ConVar c, const char[] o, const char[] n) { g_iInitialSpawnInt = g_hCvarInitialSpawn.IntValue; }
public void ConVarDirectorCvarChanged(ConVar c, const char[] o, const char[] n) { g_bDirectorCvarsModified = true; }
public void ConVarAdjustSpawnTimes(ConVar c, const char[] o, const char[] n) { g_bAdjustSpawnTimes = g_hCvarAdjustSpawnTimes.BoolValue; }
public void ConVarCoordination(ConVar c, const char[] o, const char[] n) { g_bCoordination = g_hCvarCoordination.BoolValue; }
public void ConVarDisableSpawnsTank(ConVar c, const char[] o, const char[] n) { g_bDisableSpawnsTank = g_hCvarDisableSpawnsTank.BoolValue; }

public void ConVarMaxPlayerZombies(ConVar c, const char[] o, const char[] n)
{
    g_iMaxPlayerZombies        = g_hCvarPluginMaxSpecials.IntValue;
    g_iDefaultMaxPlayerZombies = g_iMaxPlayerZombies;
    CreateTimer(0.1, Timer_MaxSpecialsSet, _, TIMER_FLAG_NO_MAPCHANGE);
}

public void ConVarDirectorSpawn(ConVar c, const char[] o, const char[] n)
{
    g_bDirectorSpawn = g_hCvarDirectorSpawn.BoolValue;

    if (g_bDirectorSpawn)
    {
        ClearAllSpawnTimers();
        StopIdleCheck();
        DirectorStuff();
    }
    else
    {
        TweakSettings();
        if (g_bLeftSaveRoom) EnableAllSpawnFlags();
        StartIdleCheck();
        CheckIfBotsNeeded(true, false);
    }
}

public void ConVarGameMode(ConVar c, const char[] o, const char[] n)
{
    GameModeCheck();
    if (!g_bDirectorSpawn) TweakSettings(); else DirectorStuff();
}

void GameModeCheck()
{
    char sGameName[64];
    g_hCvarGameMode.GetString(sGameName, sizeof(sGameName));

    if (StrEqual(sGameName, "survival", false)) g_iGameMode = GAMEMODE_SURVIVAL;
    else if (StrEqual(sGameName, "versus", false) || StrEqual(sGameName, "teamversus", false)
        || StrEqual(sGameName, "scavenge", false) || StrEqual(sGameName, "teamscavenge", false)
        || StrEqual(sGameName, "mutation12", false) || StrEqual(sGameName, "mutation13", false)
        || StrEqual(sGameName, "mutation15", false) || StrEqual(sGameName, "mutation11", false))
        g_iGameMode = GAMEMODE_VERSUS;
    else g_iGameMode = GAMEMODE_COOP;
}

// ===== Tweak / Reset =====
void TweakSettings()
{
    ResetCvars();

    switch (g_iGameMode)
    {
        case GAMEMODE_COOP:
        {
            if (g_bIsL4D2)
            {
                SetCvarInt("z_smoker_limit", 0);
                SetCvarInt("z_boomer_limit", 0);
                SetCvarInt("z_hunter_limit", 0);
                SetCvarInt("z_spitter_limit", 0);
                SetCvarInt("z_jockey_limit", 0);
                SetCvarInt("z_charger_limit", 0);
            }
            else
            {
                SetCvarInt("z_gas_limit", 0);
                SetCvarInt("z_exploding_limit", 0);
                SetCvarInt("z_hunter_limit", 0);
            }
        }
        case GAMEMODE_VERSUS:
        {
            if (g_bIsL4D2)
            {
                SetCvarInt("z_smoker_limit", 0);
                SetCvarInt("z_boomer_limit", 0);
                SetCvarInt("z_hunter_limit", 0);
                SetCvarInt("z_spitter_limit", 0);
                SetCvarInt("z_jockey_limit", 0);
                SetCvarInt("z_charger_limit", 0);
                SetCvarInt("z_jockey_leap_time", 0);
                SetCvarInt("z_spitter_max_wait_time", 0);
            }
            else
            {
                SetCvarInt("z_gas_limit", 999);
                SetCvarInt("z_exploding_limit", 999);
                SetCvarInt("z_hunter_limit", 999);
            }
            SetCvarFloat("smoker_tongue_delay", 0.0);
            SetCvarFloat("boomer_vomit_delay", 0.0);
            SetCvarFloat("boomer_exposed_time_tolerance", 0.0);
            SetCvarInt("hunter_leap_away_give_up_range", 0);
            SetCvarInt("z_hunter_lunge_distance", 5000);
            SetCvarInt("hunter_pounce_ready_range", 1500);
            SetCvarFloat("hunter_pounce_loft_rate", 0.055);
            SetCvarFloat("z_hunter_lunge_stagger_time", 0.0);
        }
        case GAMEMODE_SURVIVAL:
        {
            if (g_bIsL4D2)
            {
                SetCvarInt("survival_max_smokers", 0);
                SetCvarInt("survival_max_boomers", 0);
                SetCvarInt("survival_max_hunters", 0);
                SetCvarInt("survival_max_spitters", 0);
                SetCvarInt("survival_max_jockeys", 0);
                SetCvarInt("survival_max_chargers", 0);
                SetCvarInt("survival_max_specials", g_iMaxPlayerZombies);
                SetCvarInt("z_smoker_limit", 0);
                SetCvarInt("z_boomer_limit", 0);
                SetCvarInt("z_hunter_limit", 0);
                SetCvarInt("z_spitter_limit", 0);
                SetCvarInt("z_jockey_limit", 0);
                SetCvarInt("z_charger_limit", 0);
            }
            else
            {
                SetCvarInt("holdout_max_smokers", 0);
                SetCvarInt("holdout_max_boomers", 0);
                SetCvarInt("holdout_max_hunters", 0);
                SetCvarInt("holdout_max_specials", g_iMaxPlayerZombies);
                SetCvarInt("z_gas_limit", 0);
                SetCvarInt("z_exploding_limit", 0);
                SetCvarInt("z_hunter_limit", 0);
            }
        }
    }

    SetCvarInt("z_attack_flow_range", 50000);
    SetCvarInt("z_spawn_flow_limit", 50000);
    SetCvarInt("director_spectate_specials", 0);

    g_bDirectorCvarsModified = false;
    if (g_bIsL4D2) SetCvarInt("versus_special_respawn_interval", 99999999);
}

void ResetCvars()
{
    if (g_iGameMode == GAMEMODE_COOP)
    {
        RestoreDefaultSafe("director_no_specials");
        RestoreDefaultSafe("boomer_exposed_time_tolerance");
        RestoreDefaultSafe("hunter_pounce_ready_range");
        RestoreDefaultSafe("z_hunter_lunge_stagger_time");
        if (g_bIsL4D2)
        {
            RestoreDefaultSafe("survival_max_smokers");
            RestoreDefaultSafe("survival_max_boomers");
            RestoreDefaultSafe("survival_max_hunters");
            RestoreDefaultSafe("survival_max_spitters");
            RestoreDefaultSafe("survival_max_jockeys");
            RestoreDefaultSafe("survival_max_chargers");
            RestoreDefaultSafe("survival_max_specials");
            RestoreDefaultSafe("z_jockey_leap_time");
            RestoreDefaultSafe("z_spitter_max_wait_time");
        }
        else
        {
            RestoreDefaultSafe("holdout_max_smokers");
            RestoreDefaultSafe("holdout_max_boomers");
            RestoreDefaultSafe("holdout_max_hunters");
            RestoreDefaultSafe("holdout_max_specials");
        }
    }
    else if (g_iGameMode == GAMEMODE_VERSUS)
    {
        if (g_bIsL4D2)
        {
            RestoreDefaultSafe("survival_max_smokers");
            RestoreDefaultSafe("survival_max_boomers");
            RestoreDefaultSafe("survival_max_hunters");
            RestoreDefaultSafe("survival_max_spitters");
            RestoreDefaultSafe("survival_max_jockeys");
            RestoreDefaultSafe("survival_max_chargers");
            RestoreDefaultSafe("survival_max_specials");
        }
        else
        {
            RestoreDefaultSafe("holdout_max_smokers");
            RestoreDefaultSafe("holdout_max_boomers");
            RestoreDefaultSafe("holdout_max_hunters");
            RestoreDefaultSafe("holdout_max_specials");
        }
    }
    else if (g_iGameMode == GAMEMODE_SURVIVAL)
    {
        RestoreDefaultSafe("z_hunter_limit");
        if (g_bIsL4D2)
        {
            RestoreDefaultSafe("z_smoker_limit");
            RestoreDefaultSafe("z_boomer_limit");
            RestoreDefaultSafe("z_spitter_limit");
            RestoreDefaultSafe("z_jockey_limit");
            RestoreDefaultSafe("z_charger_limit");
            RestoreDefaultSafe("z_jockey_leap_time");
            RestoreDefaultSafe("z_spitter_max_wait_time");
        }
        else
        {
            RestoreDefaultSafe("z_gas_limit");
            RestoreDefaultSafe("z_exploding_limit");
        }
        RestoreDefaultSafe("director_no_specials");
        RestoreDefaultSafe("boomer_exposed_time_tolerance");
        RestoreDefaultSafe("hunter_pounce_ready_range");
        RestoreDefaultSafe("z_hunter_lunge_stagger_time");
    }
}

void ResetCvarsDirector()
{
    if (g_iGameMode != GAMEMODE_VERSUS)
    {
        if (g_bIsL4D2)
        {
            RestoreDefaultSafe("z_smoker_limit");
            RestoreDefaultSafe("z_boomer_limit");
            RestoreDefaultSafe("z_hunter_limit");
            RestoreDefaultSafe("z_spitter_limit");
            RestoreDefaultSafe("z_jockey_limit");
            RestoreDefaultSafe("z_charger_limit");
            RestoreDefaultSafe("survival_max_smokers");
            RestoreDefaultSafe("survival_max_boomers");
            RestoreDefaultSafe("survival_max_hunters");
            RestoreDefaultSafe("survival_max_spitters");
            RestoreDefaultSafe("survival_max_jockeys");
            RestoreDefaultSafe("survival_max_chargers");
            RestoreDefaultSafe("survival_max_specials");
        }
        else
        {
            RestoreDefaultSafe("z_hunter_limit");
            RestoreDefaultSafe("z_exploding_limit");
            RestoreDefaultSafe("z_gas_limit");
            RestoreDefaultSafe("holdout_max_smokers");
            RestoreDefaultSafe("holdout_max_boomers");
            RestoreDefaultSafe("holdout_max_hunters");
            RestoreDefaultSafe("holdout_max_specials");
        }
    }
    else
    {
        if (g_bIsL4D2)
        {
            SetCvarInt("z_smoker_limit", 2);
            RestoreDefaultSafe("z_boomer_limit");
            SetCvarInt("z_hunter_limit", 2);
            RestoreDefaultSafe("z_spitter_limit");
            RestoreDefaultSafe("z_jockey_limit");
            RestoreDefaultSafe("z_charger_limit");
        }
        else
        {
            RestoreDefaultSafe("z_hunter_limit");
            RestoreDefaultSafe("z_exploding_limit");
            RestoreDefaultSafe("z_gas_limit");
        }
    }
}

void DirectorStuff()
{
    g_bSpecialHalt = false;
    SetCvarInt("director_spectate_specials", 1);
    if (g_bIsL4D2) RestoreDefaultSafe("versus_special_respawn_interval");
    if (!g_bDirectorCvarsModified) ResetCvarsDirector();
}

// ===== 延迟上限 =====
float GetSpawnDelayCap()
{
    float incap = (g_hCvarIncapSpawnDelay != null) ? g_hCvarIncapSpawnDelay.FloatValue : 0.0;
    float tank  = (g_hCvarTankSpawnDelay  != null) ? g_hCvarTankSpawnDelay.FloatValue  : 0.0;

    float cap = g_fInfectedSpawnTimeMax + incap * 8.0 + tank + 10.0;
    if (cap < 30.0) cap = 30.0;
    if (cap > SPAWN_DELAY_CAP_HARD) cap = SPAWN_DELAY_CAP_HARD;
    return cap;
}

// ===== Slot 管理 =====
void ScheduleSpawn(float delay)
{
    float fExtra = g_hCvarIncapSpawnDelay.FloatValue;
    if (fExtra > 0.0 && g_iCurrentIncapCount > 0)
        delay += g_iCurrentIncapCount * fExtra;

    if (g_bTankBonusActive)
    {
        float fTankExtra = g_hCvarTankSpawnDelay.FloatValue;
        if (fTankExtra > 0.0) delay += fTankExtra;
    }

    float cap = GetSpawnDelayCap();
    if (delay > cap) delay = cap;
    if (delay < SPAWN_DELAY_MIN) delay = SPAWN_DELAY_MIN;

    PushSpawnTimer(delay);
}

void ScheduleSpawnRaw(float delay)
{
    if (delay > SPAWN_DELAY_CAP_HARD) delay = SPAWN_DELAY_CAP_HARD;
    if (delay < SPAWN_DELAY_MIN) delay = SPAWN_DELAY_MIN;
    PushSpawnTimer(delay);
}

void PushSpawnTimer(float delay)
{
    if (g_iInfectedBotQueue >= g_iMaxPlayerZombies)
        return;

    for (int i = 1; i <= MaxClients; i++)
    {
        if (g_hSpawnTimer[i] == null && !g_bSpawnReady[i])
        {
            g_fSpawnTimerDeadline[i] = GetGameTime() + delay;
            g_hSpawnTimer[i] = CreateTimer(delay, Timer_SpawnInfectedBot, i, TIMER_FLAG_NO_MAPCHANGE);
            g_iInfectedBotQueue++;
            return;
        }
    }

    for (int i = 1; i <= MaxClients; i++)
    {
        if (g_hSpawnTimer[i] == null && g_bSpawnReady[i])
        {
            g_bSpawnReady[i] = false;
            g_fSpawnTimerDeadline[i] = GetGameTime() + delay;
            g_hSpawnTimer[i] = CreateTimer(delay, Timer_SpawnInfectedBot, i, TIMER_FLAG_NO_MAPCHANGE);
            g_iInfectedBotQueue++;

            static float s_fStealLog = 0.0;
            float fNow = GetGameTime();
            if (fNow - s_fStealLog > 5.0)
            {
                LogMessage("[IB] Slot steal: reused ready slot %d", i);
                s_fStealLog = fNow;
            }
            return;
        }
    }

    static float s_fFullLog = 0.0;
    float fNow2 = GetGameTime();
    if (fNow2 - s_fFullLog > 5.0)
    {
        LogMessage("[IB] PushSpawnTimer: all %d slots busy, request dropped", MaxClients);
        s_fFullLog = fNow2;
    }
}

void ScheduleRetry()
{
    g_iSpawnFailStreak++;

    float d = RETRY_BASE + RETRY_STEP * float(g_iSpawnFailStreak - 1);
    if (d > RETRY_MAX) d = RETRY_MAX;
    d += GetRandomFloat(0.0, 0.5);

    ScheduleSpawnRaw(d);
}

bool TryConsumeOneReadySpawn()
{
    CountInfected();
    int currentSpecials = g_iInfectedBotCount + g_iInfectedRealCount;
    float cap = GetSpawnDelayCap();

    for (int i = 1; i <= MaxClients; i++)
    {
        if (!g_bSpawnReady[i]) continue;

        if (currentSpecials + g_iInfectedBotQueue >= g_iMaxPlayerZombies)
        {
            g_bSpawnReady[i] = false;
            continue;
        }

        g_bSpawnReady[i] = false;

        float spawnTime;
        if (g_bInitialSpawn)
        {
            spawnTime = SPAWN_FIRSTWAVE_TOPUP;
        }
        else
        {
            spawnTime = GetURandomFloatRange(g_fInfectedSpawnTimeMin, g_fInfectedSpawnTimeMax);
            if (spawnTime > cap) spawnTime = cap;
        }
        if (spawnTime < SPAWN_DELAY_MIN) spawnTime = SPAWN_DELAY_MIN;

        g_fSpawnTimerDeadline[i] = GetGameTime() + spawnTime;
        g_hSpawnTimer[i] = CreateTimer(spawnTime, Timer_SpawnInfectedBot, i, TIMER_FLAG_NO_MAPCHANGE);
        g_iInfectedBotQueue++;
        return true;
    }
    return false;
}

void ClearAllSpawnTimers()
{
    for (int i = 1; i <= MaxClients; i++)
    {
        if (g_hSpawnTimer[i] != null) { delete g_hSpawnTimer[i]; g_hSpawnTimer[i] = null; }
        g_bSpawnReady[i] = false;
        g_fSpawnTimerDeadline[i] = 0.0;
    }
    g_iInfectedBotQueue = 0;
    g_bLastSpawnIsFallback = false;
    g_iSpawnFailStreak = 0;

    for (int zc = 1; zc <= 6; zc++)
    {
        g_iClassReserved[zc] = 0;
        g_fReserveExpire[zc] = 0.0;
    }
}

void ClearClassCooldown()
{
    for (int zc = 1; zc <= 6; zc++) g_fClassCooldown[zc] = 0.0;
}

void AdjustAllSpawnTimers(float delta)
{
    if (delta == 0.0) return;

    float now = GetGameTime();
    float cap = GetSpawnDelayCap();

    for (int i = 1; i <= MaxClients; i++)
    {
        if (g_hSpawnTimer[i] == null) continue;

        float remaining = g_fSpawnTimerDeadline[i] - now;
        if (remaining < 0.05) remaining = 0.05;

        float newRemaining = remaining + delta;
        if (newRemaining < SPAWN_DELAY_MIN) newRemaining = SPAWN_DELAY_MIN;
        if (newRemaining > cap) newRemaining = cap;

        delete g_hSpawnTimer[i];
        g_fSpawnTimerDeadline[i] = now + newRemaining;
        g_hSpawnTimer[i] = CreateTimer(newRemaining, Timer_SpawnInfectedBot, i, TIMER_FLAG_NO_MAPCHANGE);
    }
}

int CountIncapacitatedSurvivors()
{
    int count = 0;
    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (IsClientInKickQueue(i)) continue;
        if (GetClientTeam(i) != TEAM_SURVIVORS) continue;
        if (!PlayerIsAlive(i)) continue;
        if (GetEntProp(i, Prop_Send, "m_isIncapacitated")) count++;
    }
    return count;
}

void ValidateIncapCount()
{
    g_iCurrentIncapCount = CountIncapacitatedSurvivors();

    for (int i = 1; i <= MaxClients; i++) g_bIsIncap[i] = false;

    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (IsClientInKickQueue(i)) continue;
        if (GetClientTeam(i) != TEAM_SURVIVORS) continue;
        if (!PlayerIsAlive(i)) continue;
        if (GetEntProp(i, Prop_Send, "m_isIncapacitated")) g_bIsIncap[i] = true;
    }
}

bool IsAnyTankAlive()
{
    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (IsClientInKickQueue(i)) continue;
        if (GetClientTeam(i) != TEAM_INFECTED) continue;
        if (!PlayerIsAlive(i)) continue;
        if (IsPlayerTank(i)) return true;
    }
    return false;
}

void ValidateTankBonus()
{
    bool tankNow = IsAnyTankAlive();
    if (tankNow == g_bTankBonusActive) return;

    float fExtra = g_hCvarTankSpawnDelay.FloatValue;
    g_bTankBonusActive = tankNow;
    if (fExtra > 0.0) AdjustAllSpawnTimers(tankNow ? fExtra : -fExtra);
}

// ===== 类型预订 / 冷却 =====
int GetClassLimit(int zc)
{
    switch (zc)
    {
        case ZC_SMOKER:  return g_iSmokerLimit;
        case ZC_BOOMER:  return g_iBoomerLimit;
        case ZC_HUNTER:  return g_iHunterLimit;
        case ZC_SPITTER: return g_iSpitterLimit;
        case ZC_JOCKEY:  return g_iJockeyLimit;
        case ZC_CHARGER: return g_iChargerLimit;
    }
    return 0;
}

int CountAliveClass(int zc)
{
    int n = 0;
    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (GetClientTeam(i) != TEAM_INFECTED) continue;
        if (!PlayerIsAlive(i)) continue;
        if (GetEntProp(i, Prop_Send, "m_zombieClass") == zc) n++;
    }
    return n;
}

void ReserveClass(int zc)
{
    if (zc < 1 || zc > 6) return;
    g_iClassReserved[zc]++;
    g_fReserveExpire[zc] = GetGameTime() + CLASS_RESERVE_TTL;
}

void ReleaseClass(int zc)
{
    if (zc < 1 || zc > 6) return;
    if (g_iClassReserved[zc] > 0) g_iClassReserved[zc]--;
}

void ExpireClassReservations()
{
    float now = GetGameTime();
    for (int i = 1; i <= 6; i++)
    {
        if (g_iClassReserved[i] > 0 && now > g_fReserveExpire[i])
            g_iClassReserved[i] = 0;
    }
}

bool CanSpawnClass(int zc)
{
    if (zc < 1 || zc > 6) return false;
    int limit = GetClassLimit(zc);
    if (limit <= 0) return false;

    if (limit <= 1 && GetGameTime() < g_fClassCooldown[zc]) return false;

    return (CountAliveClass(zc) + g_iClassReserved[zc] < limit);
}

// ===== 人数联动 =====
void RecalculateSpecialLimit()
{
    if (g_bDirectorSpawn) return;
    if (!g_bLeftSaveRoom) return;
    if (g_iGameMode == GAMEMODE_VERSUS) return;
    if (!g_hCvarAIEnabled.BoolValue) return;

    int survivors  = TotalSurvivors_();
    int aiDefault  = g_hCvarAIDefaultPlayerCount.IntValue;
    int aiPerGroup = g_hCvarAIPlayerCountOfAutoAdd.IntValue;
    int aiAddCount = g_hCvarAICountOfAutoAdd.IntValue;

    int newMax = g_iDefaultMaxPlayerZombies;
    if (survivors > aiDefault && aiPerGroup > 0)
    {
        int addGroups = (survivors - aiDefault) / aiPerGroup;
        newMax += aiAddCount * addGroups;
    }

    CountInfected();
    int alive = g_iInfectedBotCount + g_iInfectedRealCount;

    int usedAlive = 0;
    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (!IsFakeClient(i)) { usedAlive++; continue; }
        if (GetClientTeam(i) == TEAM_SURVIVORS) { usedAlive++; continue; }
        if (PlayerIsAlive(i)) usedAlive++;
    }
    int freeSlots = MaxClients - usedAlive;
    if (freeSlots < 0) freeSlots = 0;

    int slotCap = alive + freeSlots;
    if (slotCap < alive + 1) slotCap = alive + 1;
    if (newMax > slotCap) newMax = slotCap;

    int classCap = g_iSmokerLimit + g_iBoomerLimit + g_iHunterLimit;
    if (g_bIsL4D2) classCap += g_iSpitterLimit + g_iJockeyLimit + g_iChargerLimit;
    if (classCap < 1) classCap = 1;
    if (newMax > classCap) newMax = classCap;

    if (newMax != g_iMaxPlayerZombies)
    {
        g_iMaxPlayerZombies = newMax;
        g_hZMaxPlayerZombies.SetInt(g_iMaxPlayerZombies);
    }

    if (survivors != g_iLastSurvivors)
    {
        g_iLastSurvivors = survivors;
        delete g_hDebouncePrintTimer;
        g_hDebouncePrintTimer = CreateTimer(2.0, Timer_DebouncedPrint, _, TIMER_FLAG_NO_MAPCHANGE);
    }
}

public Action Timer_DebouncedPrint(Handle timer)
{
    g_hDebouncePrintTimer = null;
    if (g_iLastSurvivors < 0) return Plugin_Stop;

    int actual = TotalSurvivors_();
    if (actual != g_iLastSurvivors)
    {
        g_iLastSurvivors = actual;
        delete g_hDebouncePrintTimer;
        g_hDebouncePrintTimer = CreateTimer(2.0, Timer_DebouncedPrint, _, TIMER_FLAG_NO_MAPCHANGE);
        return Plugin_Stop;
    }

    PrintToChatAll("\x04[提示]\x05存活幸存者\x04:\x03%d\x05人\x04,\x05特感上限\x04:\x03%d\x05特\x04,\x05刷新间隔\x04:\x03%.1f\x04~\x03%.1f\x05秒.",
        g_iLastSurvivors, g_iMaxPlayerZombies, g_fInfectedSpawnTimeMin, g_fInfectedSpawnTimeMax);
    return Plugin_Stop;
}

public void DelayedRecalculate()
{
    RecalculateSpecialLimit();
    CheckIfBotsNeeded(false, false);
}

// ===== 控制幸存者检测 =====
bool IsPinningSurvivor(int client)
{
    if (client < 1 || client > MaxClients) return false;
    if (!IsClientInGame(client)) return false;
    if (GetClientTeam(client) != TEAM_INFECTED) return false;
    if (!PlayerIsAlive(client)) return false;
    if (IsPlayerGhost(client)) return false;
    if (IsPlayerTank(client)) return false;

    int zc = GetEntProp(client, Prop_Send, "m_zombieClass");
    int victim;

    switch (zc)
    {
        case ZC_SMOKER:
        {
            victim = GetEntPropEnt(client, Prop_Send, "m_tongueVictim");
            return (victim > 0 && victim <= MaxClients && IsClientInGame(victim) && PlayerIsAlive(victim));
        }
        case ZC_HUNTER:
        {
            victim = GetEntPropEnt(client, Prop_Send, "m_pounceVictim");
            return (victim > 0 && victim <= MaxClients && IsClientInGame(victim) && PlayerIsAlive(victim));
        }
        case ZC_JOCKEY:
        {
            if (!g_bIsL4D2) return false;
            victim = GetEntPropEnt(client, Prop_Send, "m_jockeyVictim");
            return (victim > 0 && victim <= MaxClients && IsClientInGame(victim) && PlayerIsAlive(victim));
        }
        case ZC_CHARGER:
        {
            if (!g_bIsL4D2) return false;
            victim = GetEntPropEnt(client, Prop_Send, "m_pummelVictim");
            if (victim > 0 && victim <= MaxClients && IsClientInGame(victim) && PlayerIsAlive(victim)) return true;
            victim = GetEntPropEnt(client, Prop_Send, "m_carryVictim");
            if (victim > 0 && victim <= MaxClients && IsClientInGame(victim) && PlayerIsAlive(victim)) return true;
            return false;
        }
    }
    return false;
}

// ===== IB 对外接口 =====
public void IB_OnInfectedTeleported(int client)
{
    if (client < 1 || client > MaxClients) return;
    if (!IsClientInGame(client)) return;
    if (GetClientTeam(client) != TEAM_INFECTED) return;

    g_fSpawnLifeTime[client] = GetGameTime();
    g_bPendingLifespanKill[client] = false;
}

public void IB_OnStuckSlain(int client)
{
    if (client < 1 || client > MaxClients) return;
    LogMessage("[IB] IB_OnStuckSlain external call (client=%d)", client);
}

// ===== 闲置 / 寿命检测 =====
void StartIdleCheck()
{
    StopIdleCheck();

    for (int i = 1; i <= MaxClients; i++)
    {
        g_fIdleTime[i] = 0.0;
        g_fLastPos[i][0] = 0.0;
        g_fLastPos[i][1] = 0.0;
        g_fLastPos[i][2] = 0.0;
        g_fSpawnLifeTime[i] = 0.0;
        g_bPendingLifespanKill[i] = false;
        g_fBotFirstSeen[i] = 0.0;
        g_fGhostSince[i] = 0.0;
    }

    g_hIdleCheckTimer = CreateTimer(IDLE_CHECK_INTERVAL, Timer_CheckIdleSpecials, _,
        TIMER_REPEAT | TIMER_FLAG_NO_MAPCHANGE);
}

void StopIdleCheck()
{
    if (g_hIdleCheckTimer != null) { delete g_hIdleCheckTimer; g_hIdleCheckTimer = null; }
}

float GetClientSpeedManual(int client)
{
    float vec[3];
    GetEntPropVector(client, Prop_Data, "m_vecVelocity", vec);
    return GetVectorLength(vec);
}

public Action Timer_CheckIdleSpecials(Handle timer)
{
    if (!g_bLeftSaveRoom || g_bHasRoundEnded) return Plugin_Continue;

    int threshold = g_hCvarIdletimeB4Slay.IntValue;
    int lifespan  = g_hCvarMaxLifetime.IntValue;
    float fGrace  = g_hCvarLifespanGrace.FloatValue;

    if (threshold <= 0 && lifespan <= 0) return Plugin_Continue;

    float fCurPos[3], fSpeed;
    float now = GetGameTime();

    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (GetClientTeam(i) != TEAM_INFECTED) continue;
        if (!PlayerIsAlive(i)) continue;
        if (!IsFakeClient(i)) continue;
        if (IsPlayerTank(i)) continue;

        if (IsPlayerGhost(i))
        {
            if (g_iGameMode == GAMEMODE_VERSUS || g_bDirectorSpawn)
            {
                g_fGhostSince[i] = 0.0;
                g_fBotFirstSeen[i] = 0.0;
                continue;
            }

            if (g_fGhostSince[i] == 0.0) g_fGhostSince[i] = now;
            else if (now - g_fGhostSince[i] > 8.0)
            {
                LogMessage("[IB] 幽灵超时回收: client=%d class=%d", i, GetEntProp(i, Prop_Send, "m_zombieClass"));
                g_fGhostSince[i] = now;
                CreateTimer(0.1, Timer_KickBot, i, TIMER_FLAG_NO_MAPCHANGE);
            }

            g_fBotFirstSeen[i] = 0.0;
            continue;
        }
        g_fGhostSince[i] = 0.0;

        if (fGrace > 0.0)
        {
            if (g_fBotFirstSeen[i] == 0.0) { g_fBotFirstSeen[i] = now; continue; }
            if (now - g_fBotFirstSeen[i] < fGrace) continue;
        }

        if (lifespan > 0)
        {
            if (g_fSpawnLifeTime[i] == 0.0) g_fSpawnLifeTime[i] = now;
            float lifeTime = now - g_fSpawnLifeTime[i];

            if (lifeTime >= float(lifespan))
            {
                if (IsPinningSurvivor(i))
                {
                    if (!g_bPendingLifespanKill[i])
                    {
                        g_bPendingLifespanKill[i] = true;
                        LogMessage("[IB] Lifespan reached but pinning, pending kill: client=%d class=%d lifeTime=%.1fs",
                            i, GetEntProp(i, Prop_Send, "m_zombieClass"), lifeTime);
                    }
                }
                else
                {
                    LogMessage("[IB] Lifespan kill: client=%d class=%d lifeTime=%.1fs",
                        i, GetEntProp(i, Prop_Send, "m_zombieClass"), lifeTime);
                    g_fSpawnLifeTime[i] = 0.0;
                    g_bPendingLifespanKill[i] = false;
                    CreateTimer(0.1, Timer_KickBot, i, TIMER_FLAG_NO_MAPCHANGE);
                    continue;
                }
            }
        }

        if (g_bPendingLifespanKill[i])
        {
            if (!IsPinningSurvivor(i))
            {
                LogMessage("[IB] Pending lifespan kill (released): client=%d class=%d",
                    i, GetEntProp(i, Prop_Send, "m_zombieClass"));
                g_fSpawnLifeTime[i] = 0.0;
                g_bPendingLifespanKill[i] = false;
                CreateTimer(0.1, Timer_KickBot, i, TIMER_FLAG_NO_MAPCHANGE);
                continue;
            }
        }

        if (threshold > 0)
        {
            if (IsPinningSurvivor(i))
            {
                GetClientAbsOrigin(i, fCurPos);
                g_fLastPos[i] = fCurPos;
                g_fIdleTime[i] = 0.0;
                continue;
            }

            GetClientAbsOrigin(i, fCurPos);
            fSpeed = GetClientSpeedManual(i);

            if (g_fLastPos[i][0] != 0.0 || g_fLastPos[i][1] != 0.0)
            {
                float fDist = GetVectorDistance(fCurPos, g_fLastPos[i]);
                if (fDist > TELEPORT_DISTANCE_THRESHOLD)
                {
                    g_fIdleTime[i] = 0.0;
                    g_fLastPos[i] = fCurPos;
                    continue;
                }
            }
            g_fLastPos[i] = fCurPos;

            if (fSpeed < IDLE_SPEED_THRESHOLD)
            {
                g_fIdleTime[i] += IDLE_CHECK_INTERVAL;
                if (g_fIdleTime[i] >= float(threshold))
                {
                    CreateTimer(0.1, Timer_KickBot, i, TIMER_FLAG_NO_MAPCHANGE);
                    g_fIdleTime[i] = 0.0;
                    g_fLastPos[i][0] = 0.0;
                    g_fLastPos[i][1] = 0.0;
                    g_fLastPos[i][2] = 0.0;
                }
            }
            else g_fIdleTime[i] = 0.0;
        }
    }
    return Plugin_Continue;
}

// ===== 坦克事件 =====
public Action evtTankSpawn(Event event, const char[] name, bool dontBroadcast)
{
    if (g_bTankBonusActive) return Plugin_Continue;
    g_bTankBonusActive = true;

    float fExtra = g_hCvarTankSpawnDelay.FloatValue;
    if (fExtra > 0.0) AdjustAllSpawnTimers(fExtra);
    return Plugin_Continue;
}

public Action evtTankKilled(Event event, const char[] name, bool dontBroadcast)
{
    if (!g_bTankBonusActive) return Plugin_Continue;
    g_bTankBonusActive = false;

    float fExtra = g_hCvarTankSpawnDelay.FloatValue;
    if (fExtra > 0.0) AdjustAllSpawnTimers(-fExtra);
    return Plugin_Continue;
}

// ===== 导演 Script Value 接管 =====
public Action L4D_OnGetScriptValueInt(const char[] key, int &retVal)
{
    if (strcmp(key, "MaxSpecials", false) == 0 || strcmp(key, "cm_MaxSpecials", false) == 0)
    {
        int val = g_iMaxPlayerZombies;
        if (val < 1)  val = 1;
        if (val > 28) val = 28;
        retVal = val;
        return Plugin_Handled;
    }

    if (strcmp(key, "AllowSpecialInfectedWithClass", false) == 0)
    {
        retVal = 1;
        return Plugin_Handled;
    }
    return Plugin_Continue;
}

// ===== Events =====
public Action evtRoundStart(Event event, const char[] name, bool dontBroadcast)
{
    if (g_bHasRoundStarted) return Plugin_Continue;

    g_bLeftSaveRoom        = false;
    g_bHasRoundEnded       = false;
    g_bHasRoundStarted     = true;
    g_bLastSpawnIsFallback = false;
    g_iLastSurvivors       = -1;

    g_bFirstWaveArmed  = false;
    g_bFirstWaveLogged = false;

    ClearClassCooldown();

    delete g_hDebouncePrintTimer;
    g_hDebouncePrintTimer = null;

    g_iCurrentIncapCount = 0;
    g_bTankBonusActive   = false;
    g_iSpawnSepCount     = 0;
    g_iSpawnSepHead      = 0;
    g_fHurtRefreshTime   = 0.0;
    g_fLastSpawnSuccess  = GetGameTime();
    g_fLastWatchdogReset = 0.0;

    for (int i = 1; i <= MaxClients; i++)
    {
        g_bIsIncap[i] = false;
        g_fSpawnLifeTime[i] = 0.0;
        g_bPendingLifespanKill[i] = false;
        g_fBotFirstSeen[i] = 0.0;
        g_fGhostSince[i] = 0.0;
    }

    GameModeCheck();

    if (g_iSavedZMaxFlags == -1)
    {
        g_iSavedZMaxFlags = g_hZMaxPlayerZombies.Flags;
        float upper = 0.0;
        g_bSavedZMaxHadUpper = g_hZMaxPlayerZombies.GetBounds(ConVarBound_Upper, upper);
        g_fSavedZMaxUpper = upper;
    }
    g_hZMaxPlayerZombies.SetBounds(ConVarBound_Upper, false);
    g_hZMaxPlayerZombies.Flags = g_iSavedZMaxFlags & ~FCVAR_NOTIFY;

    CreateTimer(0.4, Timer_MaxSpecialsSet, _, TIMER_FLAG_NO_MAPCHANGE);
    ClearAllSpawnTimers();

    g_iBotReady     = 0;
    g_bSpecialHalt  = false;
    g_bInitialSpawn = false;

    if (!g_bDirectorSpawn) TweakSettings(); else DirectorStuff();

    if (g_iGameMode != GAMEMODE_SURVIVAL)
        CreateTimer(1.0, Timer_PlayerLeftStart, _, TIMER_FLAG_NO_MAPCHANGE);

    return Plugin_Continue;
}

public Action evtRoundEnd(Event event, const char[] name, bool dontBroadcast)
{
    if (!g_bHasRoundEnded)
    {
        g_bHasRoundEnded   = true;
        g_bHasRoundStarted = false;
        g_bLeftSaveRoom    = false;
        g_bInitialSpawn    = false;
        g_bFirstWaveArmed  = false;

        ClearAllSpawnTimers();
        ClearClassCooldown();

        delete g_hDebouncePrintTimer;
        g_hDebouncePrintTimer = null;
        StopIdleCheck();

        if (g_hPeriodicCheckTimer != null) { delete g_hPeriodicCheckTimer; g_hPeriodicCheckTimer = null; }

        for (int i = 1; i <= MaxClients; i++)
        {
            g_fSpawnLifeTime[i] = 0.0;
            g_bPendingLifespanKill[i] = false;
            g_fBotFirstSeen[i] = 0.0;
            g_fGhostSince[i] = 0.0;
        }
    }
    return Plugin_Continue;
}

public Action evtPlayerDeath(Event event, const char[] name, bool dontBroadcast)
{
    if (g_bHasRoundEnded || !g_bLeftSaveRoom) return Plugin_Continue;

    int client = GetClientOfUserId(event.GetInt("userid"));
    if (!client || client < 1 || client > MaxClients) return Plugin_Continue;

    if (g_bIsIncap[client])
    {
        g_bIsIncap[client] = false;
        if (g_iCurrentIncapCount > 0) g_iCurrentIncapCount--;
    }

    if (!IsClientInGame(client)) return Plugin_Continue;

    int team = GetClientTeam(client);
    if (team == TEAM_SURVIVORS) { RecalculateSpecialLimit(); return Plugin_Continue; }
    if (team != TEAM_INFECTED) return Plugin_Continue;

    g_fIdleTime[client] = 0.0;
    g_fLastPos[client][0] = 0.0;
    g_fLastPos[client][1] = 0.0;
    g_fLastPos[client][2] = 0.0;
    g_fSpawnLifeTime[client] = 0.0;
    g_bPendingLifespanKill[client] = false;
    g_fBotFirstSeen[client] = 0.0;
    g_fGhostSince[client] = 0.0;

    bool isBot = IsFakeClient(client);

    if (isBot && !g_bDirectorSpawn)
    {
        if (!IsPlayerTank(client))
        {
            if (!TryConsumeOneReadySpawn())
            {
                CountInfected();
                int aliveNow = g_iInfectedBotCount + g_iInfectedRealCount;

                // [v2.3.4] Pre hook 防御：垂死者可能仍被计入
                if (PlayerIsAlive(client)) aliveNow--;

                if (aliveNow + g_iInfectedBotQueue < g_iMaxPlayerZombies)
                {
                    if (g_bInitialSpawn)
                    {
                        ScheduleSpawn(SPAWN_FIRSTWAVE_TOPUP);
                    }
                    else
                    {
                        float spawnTime = GetURandomFloatRange(g_fInfectedSpawnTimeMin, g_fInfectedSpawnTimeMax);
                        if (g_iGameMode == GAMEMODE_VERSUS && g_bAdjustSpawnTimes
                            && HumansOnInfected() != g_iMaxPlayerZombies)
                        {
                            int divisor = g_iMaxPlayerZombies - HumansOnInfected();
                            if (divisor > 0) spawnTime /= float(divisor);
                        }
                        ScheduleSpawn(spawnTime);
                    }
                }
            }
        }
        else CheckIfBotsNeeded(false, false);
    }

    if (IsPlayerTank(client)) CheckIfBotsNeeded(false, false);
    return Plugin_Continue;
}

public Action evtPlayerSpawn(Event event, const char[] name, bool dontBroadcast)
{
    int client = GetClientOfUserId(event.GetInt("userid"));
    if (!client || !IsClientInGame(client)) return Plugin_Continue;

    int team = GetClientTeam(client);

    if (team == TEAM_SURVIVORS)
    {
        CheckIfBotsNeeded(false, g_bInitialSpawn);
        return Plugin_Continue;
    }
    if (team != TEAM_INFECTED) return Plugin_Continue;

    g_fIdleTime[client] = 0.0;
    g_fLastPos[client][0] = 0.0;
    g_fLastPos[client][1] = 0.0;
    g_fLastPos[client][2] = 0.0;
    g_fSpawnLifeTime[client] = 0.0;
    g_bPendingLifespanKill[client] = false;
    g_fBotFirstSeen[client] = 0.0;
    g_fGhostSince[client] = 0.0;

    if (g_bDirectorSpawn && g_iGameMode != GAMEMODE_VERSUS)
    {
        int botNeeded = 0;
        if (IsPlayerSmoker(client)) botNeeded = 1;
        else if (IsPlayerBoomer(client)) botNeeded = 2;
        else if (IsPlayerHunter(client)) botNeeded = 3;
        else if (g_bIsL4D2 && IsPlayerSpitter(client)) botNeeded = 4;
        else if (g_bIsL4D2 && IsPlayerJockey(client)) botNeeded = 5;
        else if (g_bIsL4D2 && IsPlayerCharger(client)) botNeeded = 6;

        if (botNeeded > 0 && IsFakeClient(client) && !g_bSpecialHalt)
        {
            CreateTimer(0.1, Timer_KickBot, client, TIMER_FLAG_NO_MAPCHANGE);
            CreateTimer(0.2, Timer_SpawnInfectedBotDirector, botNeeded, TIMER_FLAG_NO_MAPCHANGE);
        }
    }

    if (IsFakeClient(client) && g_iGameMode == GAMEMODE_VERSUS && !IsPlayerTank(client))
        CreateTimer(0.1, Timer_SetUpBotGhost, client, TIMER_FLAG_NO_MAPCHANGE);

    return Plugin_Continue;
}

public Action evtReviveSuccess(Event event, const char[] name, bool dontBroadcast)
{
    if (g_bHasRoundEnded || !g_bLeftSaveRoom) return Plugin_Continue;

    int subject = GetClientOfUserId(event.GetInt("subject"));
    if (!subject || !IsClientInGame(subject)) return Plugin_Continue;
    if (GetClientTeam(subject) != TEAM_SURVIVORS) return Plugin_Continue;
    if (!g_bIsIncap[subject]) return Plugin_Continue;

    g_bIsIncap[subject] = false;
    if (g_iCurrentIncapCount > 0) g_iCurrentIncapCount--;
    return Plugin_Continue;
}

public Action evtPlayerIncapacitated(Event event, const char[] name, bool dontBroadcast)
{
    if (g_bHasRoundEnded || !g_bLeftSaveRoom) return Plugin_Continue;

    int client = GetClientOfUserId(event.GetInt("userid"));
    if (!client || !IsClientInGame(client)) return Plugin_Continue;
    if (GetClientTeam(client) != TEAM_SURVIVORS) return Plugin_Continue;
    if (g_bIsIncap[client]) return Plugin_Continue;

    g_bIsIncap[client] = true;
    g_iCurrentIncapCount++;
    return Plugin_Continue;
}

public Action evtPlayerTeam(Event event, const char[] name, bool dontBroadcast)
{
    int newTeam = event.GetInt("team");
    int oldTeam = event.GetInt("oldteam");

    if (oldTeam == TEAM_SURVIVORS && newTeam != TEAM_SURVIVORS)
    {
        int client = GetClientOfUserId(event.GetInt("userid"));
        if (client > 0 && client <= MaxClients && g_bIsIncap[client])
        {
            g_bIsIncap[client] = false;
            if (g_iCurrentIncapCount > 0) g_iCurrentIncapCount--;
        }
        RecalculateSpecialLimit();
    }

    if (!g_bHasRoundEnded && g_bLeftSaveRoom && g_iGameMode == GAMEMODE_VERSUS)
    {
        if (oldTeam == TEAM_INFECTED || newTeam == TEAM_INFECTED)
            CheckIfBotsNeeded(false, false);
        if (newTeam == TEAM_INFECTED)
            CreateTimer(1.0, Timer_InfectedBotBooterVersus, _, TIMER_FLAG_NO_MAPCHANGE);
    }
    return Plugin_Continue;
}

public Action evtSurvivalStart(Event event, const char[] name, bool dontBroadcast)
{
    if (g_iGameMode == GAMEMODE_SURVIVAL && !g_bLeftSaveRoom)
    {
        g_bLeftSaveRoom      = true;
        g_fLastSpawnSuccess  = GetGameTime();
        g_fLastWatchdogReset = 0.0;

        EnableAllSpawnFlags();

        g_bInitialSpawn    = true;
        g_bFirstWaveArmed  = true;
        g_bFirstWaveLogged = false;

        StartIdleCheck();
        CheckIfBotsNeeded(false, true);
        g_bFirstWaveArmed = false;

        CreateTimer(float(g_iInitialSpawnInt + 10), Timer_InitialSpawnReset, _, TIMER_FLAG_NO_MAPCHANGE);

        if (g_hPeriodicCheckTimer != null) delete g_hPeriodicCheckTimer;
        g_hPeriodicCheckTimer = CreateTimer(1.0, Timer_PeriodicCheck, _, TIMER_REPEAT | TIMER_FLAG_NO_MAPCHANGE);
    }
    return Plugin_Continue;
}

public Action evtFinaleStart(Event event, const char[] name, bool dontBroadcast)
{
    CreateTimer(1.0, Timer_CheckIfBotsNeededLater, true, TIMER_FLAG_NO_MAPCHANGE);
    return Plugin_Continue;
}

public Action evtBotReplacedPlayer(Event event, const char[] name, bool dontBroadcast)
{
    int bot = GetClientOfUserId(event.GetInt("bot"));
    if (bot > 0 && bot <= MaxClients) g_bAlreadyGhostedBot[bot] = true;
    return Plugin_Continue;
}

public Action evtPlayerFirstSpawned(Event event, const char[] name, bool dontBroadcast)
{
    int client = GetClientOfUserId(event.GetInt("userid"));
    if (!client || !IsValidEntity(client)) return Plugin_Continue;

    if (!IsFakeClient(client))
    {
        if (g_iOnlyClient > 0 && g_iOnlyClient == GetClientUserId(client))
        {
            CreateTimer(15.0, Timer_PlayerLeftStart, _, TIMER_FLAG_NO_MAPCHANGE);
            g_iOnlyClient = 0;
        }
    }

    if (g_bHasRoundEnded) return Plugin_Continue;
    if (IsFakeClient(client)) return Plugin_Continue;
    if (g_bPlayerHasEnteredStart[client]) return Plugin_Continue;

    g_bAlreadyGhosted[client] = false;
    g_bPlayerHasEnteredStart[client] = true;
    return Plugin_Continue;
}

// ===== 刷特核心 =====
void CheckIfBotsNeeded(bool spawnImmediately, bool initialSpawn)
{
    if (g_bDirectorSpawn) return;
    if (g_bHasRoundEnded || !g_bLeftSaveRoom) return;

    RecalculateSpecialLimit();
    while (TryConsumeOneReadySpawn()) {}
    CountInfected();

    int diff = g_iMaxPlayerZombies - (g_iInfectedBotCount + g_iInfectedRealCount + g_iInfectedBotQueue);
    if (diff <= 0) return;

    if (initialSpawn && !g_bFirstWaveArmed)
    {
        LogMessage("[IB] First wave top-up: survivors=%d max=%d add=%d",
            TotalSurvivors_(), g_iMaxPlayerZombies, diff);
    }

    for (int i = 0; i < diff; i++)
    {
        if (spawnImmediately) { ScheduleSpawn(0.5 + 0.3 * float(i)); continue; }

        if (initialSpawn)
        {
            float d = g_bFirstWaveArmed ? float(g_iInitialSpawnInt) : SPAWN_FIRSTWAVE_TOPUP;
            ScheduleSpawn(d + 0.3 * float(i));
            continue;
        }

        float spawnTime = GetURandomFloatRange(g_fInfectedSpawnTimeMin, g_fInfectedSpawnTimeMax);

        if (g_iGameMode == GAMEMODE_VERSUS && g_bAdjustSpawnTimes
            && HumansOnInfected() != g_iMaxPlayerZombies)
        {
            int divisor = g_iMaxPlayerZombies - HumansOnInfected();
            if (divisor > 0) ScheduleSpawn(spawnTime / float(divisor));
            else ScheduleSpawn(spawnTime);
        }
        else if (g_iGameMode == GAMEMODE_COOP && g_bAdjustSpawnTimes)
        {
            float t = spawnTime - float(TrueNumberOfSurvivors());
            if (t < 0.5) t = 0.5;
            ScheduleSpawn(t);
        }
        else ScheduleSpawn(spawnTime);
    }
}

// ===== 方向扇区 =====
static int RollOneSector()
{
    float roll = GetRandomFloat(0.0, 100.0);
    if (roll < 25.0) return SECTOR_FRONT;
    if (roll < 50.0) return SECTOR_REAR;
    if (roll < 75.0) return SECTOR_RIGHT;
    return SECTOR_LEFT;
}

static float GetRandomAngleForSector(int sector)
{
    switch (sector)
    {
        case SECTOR_FRONT: return GetRandomFloat(-45.0, 45.0);
        case SECTOR_REAR:  return GetRandomFloat(135.0, 225.0);
        case SECTOR_RIGHT: return GetRandomFloat(45.0, 135.0);
        case SECTOR_LEFT:  return GetRandomFloat(-135.0, -45.0);
    }
    return 0.0;
}

// ===== trace filter =====
static bool TraceFilter_Stuck(int entity, int contentsMask, any data)
{
    if (entity <= MaxClients || !IsValidEntity(entity)) return false;

    static char sClassName[32];
    GetEntityClassname(entity, sClassName, sizeof(sClassName));

    if (strcmp(sClassName, "infected") == 0 || strcmp(sClassName, "witch") == 0) return false;

    if (strcmp(sClassName, "env_physics_blocker") == 0)
    {
        if (HasEntProp(entity, Prop_Data, "m_nBlockType"))
        {
            int blockType = GetEntProp(entity, Prop_Data, "m_nBlockType");
            if (blockType == 1 || blockType == 2) return false;
        }
    }
    return true;
}

static bool TraceFilter_Visibility(int entity, int contentsMask, any data)
{
    if (entity <= MaxClients || !IsValidEntity(entity)) return false;

    static char sClassName[32];
    GetEntityClassname(entity, sClassName, sizeof(sClassName));

    if (strcmp(sClassName, "infected") == 0 || strcmp(sClassName, "witch") == 0) return false;
    return true;
}

bool WillStuck(float pos[3])
{
    static const float mins[3] = { -20.0, -20.0, 0.0 };
    static const float maxs[3] = {  20.0,  20.0, 71.0 };

    TR_TraceHullFilter(pos, pos, mins, maxs, MASK_PLAYERSOLID, TraceFilter_Stuck, 0);
    return TR_DidHit();
}

bool RayClear(float src[3], float dst[3], int mask)
{
    TR_TraceRayFilter(src, dst, mask, RayType_EndPoint, TraceFilter_Visibility, 0);
    return !TR_DidHit() || TR_GetFraction() >= 0.99;
}

static bool TraceDownToGround(const float start[3], float outEnd[3], int &traceCount)
{
    float s[3];

    for (int i = 0; i < sizeof(g_fDownOffsets); i++)
    {
        traceCount++;

        s[0] = start[0];
        s[1] = start[1];
        s[2] = start[2] + g_fDownOffsets[i];

        TR_TraceRay(s, RAY_DOWN_ANGLE, MASK_PLAYERSOLID, RayType_Infinite);

        if (TR_StartSolid()) continue;
        if (!TR_DidHit())    return false;

        TR_GetEndPosition(outEnd);
        return true;
    }
    return false;
}

// ===== 视野检测 =====
bool IsPosVisibleToAnySurvivor(float pos[3], int traceStartTick, int budgetMs)
{
    float chest[3], legs[3];

    chest[0] = pos[0]; chest[1] = pos[1]; chest[2] = pos[2] + 40.0;
    legs[0]  = pos[0]; legs[1]  = pos[1]; legs[2]  = pos[2] + 20.0;

    int visMask = (MASK_VISIBLE & MASK_SHOT);

    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (GetClientTeam(i) != TEAM_SURVIVORS) continue;
        if (!PlayerIsAlive(i)) continue;

        float eyes[3];
        GetClientEyePosition(i, eyes);

        if (RayClear(eyes, chest, visMask)) return true;
        if (GetSysTickCount() - traceStartTick >= budgetMs) return false;

        if (RayClear(eyes, legs, visMask)) return true;
        if (GetSysTickCount() - traceStartTick >= budgetMs) return false;
    }
    return false;
}

bool IsPredictedLandingVisible(float pos[3], int traceStartTick)
{
    if (IsPosVisibleToAnySurvivor(pos, traceStartTick, SPAWN_MAX_TRACE_TIME_MS)) return true;

    float start[3], end[3];
    start[0] = pos[0]; start[1] = pos[1]; start[2] = pos[2] + 4.0;
    end[0]   = pos[0]; end[1]   = pos[1]; end[2]   = pos[2] - 32.0;

    TR_TraceRayFilter(start, end, MASK_PLAYERSOLID, RayType_EndPoint, TraceFilter_Visibility, 0);

    if (TR_DidHit() && !TR_StartSolid())
    {
        float landing2[3];
        TR_GetEndPosition(landing2);
        if (IsPosVisibleToAnySurvivor(landing2, traceStartTick, SPAWN_MAX_TRACE_TIME_MS)) return true;
    }

    for (float offset = 2.0; offset <= 6.0; offset += 2.0)
    {
        float landing3[3];
        landing3[0] = pos[0];
        landing3[1] = pos[1];
        landing3[2] = pos[2] - offset;
        if (IsPosVisibleToAnySurvivor(landing3, traceStartTick, SPAWN_MAX_TRACE_TIME_MS)) return true;
    }
    return false;
}

bool IsValidSpawnNavFlags(int flags, bool finaleArea)
{
    if (!flags) return true;
    if (finaleArea && (flags & IB_NAV_SPAWN_FINALE) == 0) return false;
    return (flags & (IB_NAV_SPAWN_RESCUE_CLOSET | IB_NAV_SPAWN_RESCUE_VEHICLE | IB_NAV_SPAWN_CHECKPOINT)) == 0;
}

stock bool IsInSafeRoom(int client)
{
    float pos[3];
    GetClientAbsOrigin(client, pos);

    Address nav = L4D2Direct_GetTerrorNavArea(pos);
    if (nav == Address_Null) return false;

    return (L4D_GetNavArea_SpawnAttributes(nav) & NAV_SPAWN_CHECKPOINT) != 0;
}

// ===== 分散度 =====
bool PassMinSeparation(float pos[3], float now)
{
    float radius = g_hCvarSpawnSepRadius.FloatValue;
    if (radius <= 0.0) return true;

    float ttl = g_hCvarSpawnSepTTL.FloatValue;
    float r2 = radius * radius;

    for (int i = 0; i < g_iSpawnSepCount; i++)
    {
        if (now - g_fSpawnSepTime[i] > ttl) continue;
        if (GetVectorDistance(pos, g_fSpawnSepPos[i], true) < r2) return false;
    }
    return true;
}

void RememberSpawn(float pos[3], float now)
{
    int idx = g_iSpawnSepHead;
    g_fSpawnSepPos[idx][0] = pos[0];
    g_fSpawnSepPos[idx][1] = pos[1];
    g_fSpawnSepPos[idx][2] = pos[2];
    g_fSpawnSepTime[idx] = now;

    g_iSpawnSepHead = (idx + 1) % SPAWN_SEP_MAX_POINTS;
    if (g_iSpawnSepCount < SPAWN_SEP_MAX_POINTS) g_iSpawnSepCount++;
}

// ===== hurt trigger =====
bool IsSpawnPointInHurtTrigger(float pos[3])
{
    float now = GetGameTime();

    if (g_hHurtTriggers == null || now - g_fHurtRefreshTime > 3.0)
    {
        if (g_hHurtTriggers == null) g_hHurtTriggers = new ArrayList();
        else g_hHurtTriggers.Clear();

        int ent = -1;
        while ((ent = FindEntityByClassname(ent, "trigger_hurt")) != -1)
            g_hHurtTriggers.Push(EntIndexToEntRef(ent));

        ent = -1;
        while ((ent = FindEntityByClassname(ent, "trigger_hurt_ghost")) != -1)
            g_hHurtTriggers.Push(EntIndexToEntRef(ent));

        g_fHurtRefreshTime = now;
    }

    if (g_hHurtTriggers.Length == 0) return false;

    for (int i = 0; i < g_hHurtTriggers.Length; i++)
    {
        int ent = EntRefToEntIndex(g_hHurtTriggers.Get(i));
        if (ent <= MaxClients || !IsValidEntity(ent)) continue;
        if (HasEntProp(ent, Prop_Data, "m_bDisabled") && GetEntProp(ent, Prop_Data, "m_bDisabled") != 0) continue;

        float origin[3], tmin[3], tmax[3];
        GetEntPropVector(ent, Prop_Data, "m_vecAbsOrigin", origin);
        GetEntPropVector(ent, Prop_Send, "m_vecMins", tmin);
        GetEntPropVector(ent, Prop_Send, "m_vecMaxs", tmax);

        bool overlap = true;
        for (int axis = 0; axis < 3; axis++)
        {
            float lo = origin[axis] + tmin[axis];
            float hi = origin[axis] + tmax[axis];
            if (pos[axis] + 16.0 < lo || pos[axis] - 16.0 > hi) { overlap = false; break; }
        }
        if (overlap) return true;
    }
    return false;
}

// ===== 幸存者选择 =====
bool GetRandomAliveSurvivor(int &outClient)
{
    int survivors[MAXPLAYERS + 1];
    int count = 0;
    int safeSurvivors[MAXPLAYERS + 1];
    int safeCount = 0;

    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (GetClientTeam(i) != TEAM_SURVIVORS) continue;
        if (!PlayerIsAlive(i)) continue;

        if (GetEntProp(i, Prop_Send, "m_isIncapacitated")) continue;
        if (GetEntProp(i, Prop_Send, "m_isHangingFromLedge") != 0) continue;
        if (GetEntPropEnt(i, Prop_Send, "m_tongueOwner") > 0) continue;
        if (GetEntPropEnt(i, Prop_Send, "m_pounceAttacker") > 0) continue;

        if (g_bIsL4D2)
        {
            if (GetEntProp(i, Prop_Send, "m_isFallingFromLedge") != 0) continue;
            if (GetEntPropEnt(i, Prop_Send, "m_jockeyAttacker") > 0) continue;
            if (GetEntPropEnt(i, Prop_Send, "m_pummelAttacker") > 0) continue;
            if (GetEntPropEnt(i, Prop_Send, "m_carryAttacker") > 0) continue;
        }

        if (IsInSafeRoom(i)) safeSurvivors[safeCount++] = i;
        else survivors[count++] = i;
    }

    if (count > 0)     { outClient = survivors[GetURandomIntRange(0, count - 1)];         return true; }
    if (safeCount > 0) { outClient = safeSurvivors[GetURandomIntRange(0, safeCount - 1)]; return true; }

    int relaxed[MAXPLAYERS + 1];
    int rCount = 0;

    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (GetClientTeam(i) != TEAM_SURVIVORS) continue;
        if (!PlayerIsAlive(i)) continue;
        relaxed[rCount++] = i;
    }

    if (rCount > 0)
    {
        outClient = relaxed[GetURandomIntRange(0, rCount - 1)];

        static float s_fRelaxedLog = 0.0;
        float fNow = GetGameTime();
        if (fNow - s_fRelaxedLog > 5.0)
        {
            LogMessage("[IB] 严格筛选全员被过滤，启用放宽兜底 (surv=%d)", outClient);
            s_fRelaxedLog = fNow;
        }
        return true;
    }
    return false;
}

static bool IsSafeDistanceToAllSurvivors(float pos[3], float minDist)
{
    float r2 = minDist * minDist;

    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (GetClientTeam(i) != TEAM_SURVIVORS) continue;
        if (!PlayerIsAlive(i)) continue;

        float survPos[3];
        GetClientAbsOrigin(i, survPos);
        if (GetVectorDistance(pos, survPos, true) < r2) return false;
    }
    return true;
}

static int SectorOfAngle(float angle)
{
    if (angle < 45.0 || angle >= 315.0)       return SECTOR_FRONT;
    else if (angle >= 45.0 && angle < 135.0)  return SECTOR_RIGHT;
    else if (angle >= 135.0 && angle < 225.0) return SECTOR_REAR;
    return SECTOR_LEFT;
}

// =====================================================
// 主找位：① 快速采样(提前退出) → ② 复用可见点 → ③ 远距采样
// =====================================================
bool GetRandomSpawnPosition(int survivor, float outPos[3], int traceStartTick)
{
    if (survivor < 1 || survivor > MaxClients) return false;
    if (!IsClientInGame(survivor)) return false;

    static float s_fFailLog = 0.0;
    static float s_fOkLog   = 0.0;

    int globalTick = GetSysTickCount();

    float survPos[3], survAng[3], fwd[3], right[3];
    GetClientAbsOrigin(survivor, survPos);
    GetClientEyeAngles(survivor, survAng);
    survAng[0] = 0.0;
    survAng[2] = 0.0;
    GetAngleVectors(survAng, fwd, right, NULL_VECTOR);

    Address navSurv = L4D2Direct_GetTerrorNavArea(survPos);
    if (navSurv == Address_Null)
        navSurv = L4D_GetNearestNavArea(survPos, 120.0, false, false, false, TEAM_INFECTED);
    if (navSurv == Address_Null) return false;

    float fMin = g_hCvarSpawnMinRange.FloatValue;
    float fMax = g_hCvarSpawnMaxRange.FloatValue;
    if (fMax <= fMin) fMax = fMin + 1000.0;

    float maxScan = g_hCvarSpawnMaxScanDist.FloatValue;
    if (maxScan > 0.0 && fMax > fMin + maxScan)
        fMax = fMin + maxScan;

    // 可视生成距离不得低于普通不可视扫描上限。
    float visMin = g_hCvarSpawnVisMinRange.FloatValue;
    if (visMin < fMax) visMin = fMax;

    // 仅扩展②的候选采样范围；普通①仍严格限制在 fMin~fMax。
    float visScanMax = fMax + VIS_SCAN_EXTEND;
    if (maxScan > 0.0 && visScanMax > fMin + maxScan)
        visScanMax = fMin + maxScan;

    bool finaleArea = false;
    if (g_bIsL4D2) finaleArea = L4D_IsMissionFinalMap() && L4D2_GetCurrentFinaleStage() < 18;
    else           finaleArea = L4D_IsMissionFinalMap();

    float now = GetGameTime();
    float pathCostLimit = fMax * 1.3;
    float navDistLimit  = fMax * 1.5;

    int dirCount = RoundToNearest(360.0 / SCAN_ANGLE_STEP);
    if (dirCount < 4) dirCount = 4;

    // ---------- ① 原版 360° 近距扫描：435~1250，优先不可视 ----------
    float candPos[SCAN_MAX_CAND][3];
    float candDist[SCAN_MAX_CAND];
    Address candNav[SCAN_MAX_CAND];
    int candSector[SCAN_MAX_CAND];
    int candCount = 0;
    int totalTraces = 0;

    int cheapTick = GetSysTickCount();

    float rings[SCAN_NEAR_RINGS];
    for (int r = 0; r < SCAN_NEAR_RINGS; r++)
        rings[r] = fMin + (fMax - fMin) * (float(r + 1) / float(SCAN_NEAR_RINGS));

    for (int d = 0; d < dirCount && candCount < SCAN_MAX_CAND; d++)
    {
        bool cheapDone = false;
        float angle = float(d) * SCAN_ANGLE_STEP;
        float rad = angle * FLOAT_PI / 180.0;
        float cosA = Cosine(rad);
        float sinA = Sine(rad);
        float dirX = fwd[0] * cosA + right[0] * sinA;
        float dirY = fwd[1] * cosA + right[1] * sinA;

        for (int r = 0; r < SCAN_NEAR_RINGS && candCount < SCAN_MAX_CAND; r++)
        {
            if (totalTraces >= SCAN_MAX_TRACES ||
                (totalTraces >= SCAN_MIN_TRACES && GetSysTickCount() - cheapTick >= SCAN_CHEAP_MS) ||
                GetSysTickCount() - globalTick >= SCAN_GLOBAL_MS)
            {
                cheapDone = true;
                break;
            }

            float dist = rings[r] + GetRandomFloat(-40.0, 40.0);
            if (dist < fMin) dist = fMin;
            if (dist > fMax) dist = fMax;

            float start[3];
            start[0] = survPos[0] + dirX * dist;
            start[1] = survPos[1] + dirY * dist;
            start[2] = survPos[2] + 100.0;

            float endPos[3];
            if (!TraceDownToGround(start, endPos, totalTraces)) continue;

            float actualDist = GetVectorDistance(endPos, survPos);
            if (actualDist < fMin || actualDist > fMax) continue;

            float navPos[3];
            navPos[0] = endPos[0];
            navPos[1] = endPos[1];
            navPos[2] = endPos[2] + NAV_HEIGHT;

            Address nav = L4D2Direct_GetTerrorNavArea(navPos);
            if (nav == Address_Null) continue;
            if (!IsValidSpawnNavFlags(L4D_GetNavArea_SpawnAttributes(nav), finaleArea)) continue;

            float checkPos[3];
            checkPos[0] = endPos[0];
            checkPos[1] = endPos[1];
            checkPos[2] = endPos[2] + 2.0;

            if (WillStuck(checkPos)) continue;
            if (!IsSafeDistanceToAllSurvivors(checkPos, SPAWN_FALLBACK_SAFE_DIST)) continue;
            if (!PassMinSeparation(checkPos, now)) continue;
            if (IsSpawnPointInHurtTrigger(checkPos)) continue;

            candPos[candCount] = checkPos;
            candDist[candCount] = actualDist;
            candNav[candCount] = nav;
            candSector[candCount] = SectorOfAngle(angle);
            candCount++;
        }

        if (cheapDone) break;
    }

    // 打乱候选顺序，避免固定方向优先。
    for (int i = candCount - 1; i > 0; i--)
    {
        int j = GetURandomIntRange(0, i);
        float tp[3];
        tp = candPos[i]; candPos[i] = candPos[j]; candPos[j] = tp;
        float td = candDist[i]; candDist[i] = candDist[j]; candDist[j] = td;
        Address tn = candNav[i]; candNav[i] = candNav[j]; candNav[j] = tn;
        int ts = candSector[i]; candSector[i] = candSector[j]; candSector[j] = ts;
    }

    // ---------- ① 昂贵校验：不可视点优先 ----------
    int visPend[SCAN_MAX_CAND];
    int visPendCount = 0;
    int pathNear = 0;
    int verifyTick = GetSysTickCount();

    for (int i = 0; i < candCount; i++)
    {
        if (pathNear >= SCAN_PATH_NEAR ||
            GetSysTickCount() - verifyTick >= SCAN_VERIFY_MS ||
            GetSysTickCount() - globalTick >= SCAN_GLOBAL_MS)
            break;

        float checkPos[3];
        checkPos = candPos[i];

        bool visible = IsPosVisibleToAnySurvivor(checkPos, verifyTick, SCAN_VERIFY_MS);
        if (!visible && GetSysTickCount() - verifyTick >= SCAN_VERIFY_MS)
            break;

        if (visible)
        {
            // 近距可视点只有达到 visMin 才可进入②；默认不会低于1250。
            if (visPendCount < SCAN_MAX_CAND && candDist[i] >= visMin)
                visPend[visPendCount++] = i;
            continue;
        }

        if (!IsSafeDistanceToAllSurvivors(checkPos, fMin)) continue;

        pathNear++;
        if (!L4D2_NavAreaBuildPath(candNav[i], navSurv, pathCostLimit, TEAM_INFECTED, false)) continue;

        float navDist = L4D2_NavAreaTravelDistance(checkPos, survPos, false);
        if (navDist > 0.0 && navDist > navDistLimit) continue;

        if (IsPredictedLandingVisible(checkPos, verifyTick)) continue;

        outPos = candPos[i];
        g_bLastSpawnIsFallback = false;

        if (now - s_fOkLog > 1.0)
        {
            LogMessage("[IB] Spawn OK(invis): dist=%.0f sector=%d cand=%d traces=%d path=%d elapsed=%dms",
                candDist[i], candSector[i], candCount, totalTraces, pathNear,
                GetSysTickCount() - globalTick);
            s_fOkLog = now;
        }
        return true;
    }

    // ---------- ② 扩展候选扫描：只为可视兜底服务 ----------
    // 仅当①没有找到不可视点时执行；最多12个方向、最多36条落地射线，目标预算1ms。
    float extPos[VIS_SCAN_MAX_POINTS][3];
    Address extNav[VIS_SCAN_MAX_POINTS];
    float extDist[VIS_SCAN_MAX_POINTS];
    int extSector[VIS_SCAN_MAX_POINTS];
    int extCount = 0;
    int extTraces = 0;

    float extMin = visMin;
    if (extMin < fMax) extMin = fMax;

    int extTick = GetSysTickCount();
    bool canRunExt = (visScanMax > extMin &&
                      GetSysTickCount() - globalTick < SCAN_GLOBAL_MS - SCAN_VIS_MS);

    if (canRunExt)
    {
        float angleOffset = GetRandomFloat(0.0, 360.0 / float(VIS_SCAN_DIRS));

        for (int d = 0; d < VIS_SCAN_DIRS && extCount < VIS_SCAN_MAX_POINTS; d++)
        {
            if (extTraces >= VIS_SCAN_EXTRA_MAX_TRACES ||
                GetSysTickCount() - extTick >= VIS_SCAN_EXTRA_MS ||
                GetSysTickCount() - globalTick >= SCAN_GLOBAL_MS - SCAN_VIS_MS)
                break;

            float angle = angleOffset + float(d) * (360.0 / float(VIS_SCAN_DIRS));
            float rad = angle * FLOAT_PI / 180.0;
            float dirX = fwd[0] * Cosine(rad) + right[0] * Sine(rad);
            float dirY = fwd[1] * Cosine(rad) + right[1] * Sine(rad);
            float dist = GetRandomFloat(extMin, visScanMax);

            float start[3];
            start[0] = survPos[0] + dirX * dist;
            start[1] = survPos[1] + dirY * dist;
            start[2] = survPos[2] + 150.0;

            float endPos[3];
            if (!TraceDownToGround(start, endPos, extTraces)) continue;

            float actualDist = GetVectorDistance(endPos, survPos);
            if (actualDist < extMin || actualDist > visScanMax) continue;

            float navPos[3];
            navPos[0] = endPos[0];
            navPos[1] = endPos[1];
            navPos[2] = endPos[2] + NAV_HEIGHT;

            Address nav = L4D2Direct_GetTerrorNavArea(navPos);
            if (nav == Address_Null) continue;
            if (!IsValidSpawnNavFlags(L4D_GetNavArea_SpawnAttributes(nav), finaleArea)) continue;

            float checkPos[3];
            checkPos[0] = endPos[0];
            checkPos[1] = endPos[1];
            checkPos[2] = endPos[2] + 2.0;

            if (WillStuck(checkPos)) continue;
            if (!IsSafeDistanceToAllSurvivors(checkPos, SPAWN_FALLBACK_SAFE_DIST)) continue;
            if (!PassMinSeparation(checkPos, now)) continue;
            if (IsSpawnPointInHurtTrigger(checkPos)) continue;

            extPos[extCount] = checkPos;
            extNav[extCount] = nav;
            extDist[extCount] = actualDist;
            extSector[extCount] = SectorOfAngle(angle);
            extCount++;
        }
    }

    // 扩展候选也随机打乱，②不会总是优先同一方向。
    for (int i = extCount - 1; i > 0; i--)
    {
        int j = GetURandomIntRange(0, i);
        float tp[3];
        tp = extPos[i]; extPos[i] = extPos[j]; extPos[j] = tp;
        Address tn = extNav[i]; extNav[i] = extNav[j]; extNav[j] = tn;
        float td = extDist[i]; extDist[i] = extDist[j]; extDist[j] = td;
        int ts = extSector[i]; extSector[i] = extSector[j]; extSector[j] = ts;
    }

    // ---------- ② 可视兜底：先检查1250~1550扩展候选，再检查原扫描中达标的可视候选 ----------
    int pathVis = 0;
    int visTick = GetSysTickCount();
    float visPathCostLimit = visScanMax * 1.3;
    float visNavDistLimit = visScanMax * 1.5;

    for (int i = 0; i < extCount; i++)
    {
        if (pathVis >= SCAN_PATH_VIS ||
            GetSysTickCount() - visTick >= SCAN_VIS_MS ||
            GetSysTickCount() - globalTick >= SCAN_GLOBAL_MS)
            break;

        float checkPos[3];
        checkPos = extPos[i];

        int pointVisTick = GetSysTickCount();
        bool visible = IsPosVisibleToAnySurvivor(checkPos, pointVisTick, SCAN_VIS_MS);
        if (!visible)
        {
            if (GetSysTickCount() - visTick >= SCAN_VIS_MS) break;
            continue;
        }

        pathVis++;
        if (!L4D2_NavAreaBuildPath(extNav[i], navSurv, visPathCostLimit, TEAM_INFECTED, false)) continue;

        float navDist = L4D2_NavAreaTravelDistance(checkPos, survPos, false);
        if (navDist > 0.0 && navDist > visNavDistLimit) continue;

        outPos = extPos[i];
        g_bLastSpawnIsFallback = true;

        if (now - s_fOkLog > 1.0)
        {
            LogMessage("[IB] Spawn OK(near-vis-ext): dist=%.0f sector=%d pick=%d/%d traces=%d path=%d elapsed=%dms",
                extDist[i], extSector[i], i + 1, extCount, extTraces, pathVis,
                GetSysTickCount() - globalTick);
            s_fOkLog = now;
        }
        return true;
    }

    for (int k = 0; k < visPendCount; k++)
    {
        if (pathVis >= SCAN_PATH_VIS ||
            GetSysTickCount() - visTick >= SCAN_VIS_MS ||
            GetSysTickCount() - globalTick >= SCAN_GLOBAL_MS)
            break;

        int i = visPend[k];
        float checkPos[3];
        checkPos = candPos[i];

        pathVis++;
        if (!L4D2_NavAreaBuildPath(candNav[i], navSurv, visPathCostLimit, TEAM_INFECTED, false)) continue;

        float navDist = L4D2_NavAreaTravelDistance(checkPos, survPos, false);
        if (navDist > 0.0 && navDist > visNavDistLimit) continue;

        outPos = candPos[i];
        g_bLastSpawnIsFallback = true;

        if (now - s_fOkLog > 1.0)
        {
            LogMessage("[IB] Spawn OK(near-vis): dist=%.0f pick=%d/%d path=%d elapsed=%dms",
                candDist[i], k + 1, visPendCount, pathVis, GetSysTickCount() - globalTick);
            s_fOkLog = now;
        }
        return true;
    }

    // ---------- ③ 原版远距360°兜底：保留1250~1850范围，不更改其生成逻辑 ----------
    float farPos[SCAN_MAX_FAR_CAND][3];
    float farDist[SCAN_MAX_FAR_CAND];
    Address farNav[SCAN_MAX_FAR_CAND];
    int farCount = 0;
    int farTraces = 0;

    float farMin = fMax;
    float farMax = farMin + SCAN_FAR_EXT;
    int farTick = GetSysTickCount();

    float farRings[SCAN_FAR_RINGS];
    for (int r = 0; r < SCAN_FAR_RINGS; r++)
        farRings[r] = farMin + (farMax - farMin) * (float(r + 1) / float(SCAN_FAR_RINGS));

    for (int d = 0; d < dirCount && farCount < SCAN_MAX_FAR_CAND; d++)
    {
        bool farDone = false;
        float rad = (float(d) * SCAN_ANGLE_STEP) * FLOAT_PI / 180.0;
        float cosA = Cosine(rad);
        float sinA = Sine(rad);
        float dirX = fwd[0] * cosA + right[0] * sinA;
        float dirY = fwd[1] * cosA + right[1] * sinA;

        for (int r = 0; r < SCAN_FAR_RINGS && farCount < SCAN_MAX_FAR_CAND; r++)
        {
            if (farTraces >= SCAN_FAR_MAX_TRACES ||
                (farTraces >= 4 && GetSysTickCount() - farTick >= SCAN_FAR_MS) ||
                GetSysTickCount() - globalTick >= SCAN_GLOBAL_MS)
            {
                farDone = true;
                break;
            }

            float dist = farRings[r] + GetRandomFloat(-50.0, 50.0);
            if (dist < farMin) dist = farMin;
            if (dist > farMax) dist = farMax;

            float start[3];
            start[0] = survPos[0] + dirX * dist;
            start[1] = survPos[1] + dirY * dist;
            start[2] = survPos[2] + 200.0;

            float endPos[3];
            if (!TraceDownToGround(start, endPos, farTraces)) continue;

            float dx = endPos[0] - survPos[0];
            float dy = endPos[1] - survPos[1];
            float horiz = SquareRoot(dx * dx + dy * dy);
            if (horiz < farMin || horiz > farMax) continue;
            if (FloatAbs(endPos[2] - survPos[2]) > 600.0) continue;

            float navPos[3];
            navPos[0] = endPos[0];
            navPos[1] = endPos[1];
            navPos[2] = endPos[2] + NAV_HEIGHT;
            Address nav = L4D2Direct_GetTerrorNavArea(navPos);
            if (nav == Address_Null) continue;

            float checkPos[3];
            checkPos[0] = endPos[0];
            checkPos[1] = endPos[1];
            checkPos[2] = endPos[2] + 2.0;

            if (WillStuck(checkPos)) continue;
            if (!IsSafeDistanceToAllSurvivors(checkPos, SPAWN_FALLBACK_SAFE_DIST)) continue;
            if (!PassMinSeparation(checkPos, now)) continue;
            if (IsSpawnPointInHurtTrigger(checkPos)) continue;

            farPos[farCount] = checkPos;
            farDist[farCount] = horiz;
            farNav[farCount] = nav;
            farCount++;
        }

        if (farDone) break;
    }

    int pathFar = 0;
    if (farCount > 0)
    {
        for (int i = farCount - 1; i > 0; i--)
        {
            int j = GetURandomIntRange(0, i);
            float tp[3];
            tp = farPos[i]; farPos[i] = farPos[j]; farPos[j] = tp;
            float td = farDist[i]; farDist[i] = farDist[j]; farDist[j] = td;
            Address tn = farNav[i]; farNav[i] = farNav[j]; farNav[j] = tn;
        }

        int farIdx[SCAN_MAX_PICK];
        int farValid = 0;
        int farVerifyTick = GetSysTickCount();

        for (int i = 0; i < farCount; i++)
        {
            if (pathFar >= SCAN_PATH_FAR ||
                GetSysTickCount() - farVerifyTick >= SCAN_FAR_MS ||
                GetSysTickCount() - globalTick >= SCAN_GLOBAL_MS)
                break;

            pathFar++;
            if (!L4D2_NavAreaBuildPath(farNav[i], navSurv, farMax * 1.3, TEAM_INFECTED, false)) continue;

            farIdx[farValid++] = i;
            if (farValid >= SCAN_MAX_PICK) break;
        }

        if (farValid > 0)
        {
            int pick = farIdx[GetURandomIntRange(0, farValid - 1)];
            outPos = farPos[pick];
            g_bLastSpawnIsFallback = true;

            if (now - s_fOkLog > 1.0)
            {
                LogMessage("[IB] Spawn OK(far-vis): dist=%.0f pick=%d/%d traces=%d path=%d elapsed=%dms",
                    farDist[pick], farValid, farCount, farTraces, pathFar,
                    GetSysTickCount() - globalTick);
                s_fOkLog = now;
            }
            return true;
        }
    }

    if (now - s_fFailLog > 3.0)
    {
        LogMessage("[IB] Spawn FAIL: near=%d visPend=%d ext=%d far=%d | traces=%d/%d extTr=%d path=%d/%d elapsed=%dms total=%dms",
            candCount, visPendCount, extCount, farCount,
            totalTraces, farTraces, extTraces, pathNear, pathVis,
            GetSysTickCount() - globalTick, GetSysTickCount() - traceStartTick);
        s_fFailLog = now;
    }
    return false;
}

// ===== 刷特成功后统一处理 =====
void AcceptSpawnAndScheduleNext(int spawnedClient)
{
    if (spawnedClient < 1 || spawnedClient > MaxClients) return;
    if (!IsClientInGame(spawnedClient)) return;
    if (GetClientTeam(spawnedClient) != TEAM_INFECTED) return;
    if (!PlayerIsAlive(spawnedClient)) return;

    g_fLastSpawnSuccess = GetGameTime();

    float actualPos[3];
    GetClientAbsOrigin(spawnedClient, actualPos);
    RememberSpawn(actualPos, GetGameTime());

    if (g_bInitialSpawn)
    {
        CountInfected();
        int alive = g_iInfectedBotCount + g_iInfectedRealCount;

        if (alive >= g_iMaxPlayerZombies)
        {
            if (!g_bFirstWaveLogged)
            {
                g_bFirstWaveLogged = true;
                LogMessage("[IB] First wave filled (alive=%d max=%d), window still open", alive, g_iMaxPlayerZombies);
            }
        }
        else
        {
            CreateTimer(0.5, Timer_CheckIfBotsNeededLater, true, TIMER_FLAG_NO_MAPCHANGE);
        }
    }
    else
    {
        CreateTimer(1.0, Timer_CheckIfBotsNeededLater, false, TIMER_FLAG_NO_MAPCHANGE);
    }
}

// ===== 刷特主流程 =====
public Action Timer_SpawnInfectedBot(Handle timer, any slot)
{
    if (g_iInfectedBotQueue > 0) g_iInfectedBotQueue--;

    if (slot >= 1 && slot <= MaxClients)
    {
        g_hSpawnTimer[slot] = null;
        g_fSpawnTimerDeadline[slot] = 0.0;
    }

    if (g_bHasRoundEnded || !g_bHasRoundStarted || !g_bLeftSaveRoom)
        return Plugin_Stop;

    if (g_bDirectorSpawn) return Plugin_Stop;

    g_bLastSpawnIsFallback = false;
    int infected = g_iMaxPlayerZombies;

    if (g_bCoordination && !g_bDirectorSpawn && !g_bInitialSpawn)
    {
        g_iBotReady++;
        for (int i = 1; i <= MaxClients; i++)
        {
            if (!IsClientInGame(i)) continue;
            if (GetClientTeam(i) == TEAM_INFECTED && !IsFakeClient(i)) infected--;
        }

        if (g_iBotReady >= infected) g_iBotReady = 0;
        else
        {
            if (slot >= 1 && slot <= MaxClients) g_bSpawnReady[slot] = true;
            return Plugin_Stop;
        }
    }

    CountInfected();
    int currentSpecials = g_iInfectedRealCount + g_iInfectedBotCount;

    if (currentSpecials >= g_iMaxPlayerZombies)
    {
        if (slot >= 1 && slot <= MaxClients) g_bSpawnReady[slot] = true;
        return Plugin_Stop;
    }

    int availableSlots = g_iMaxPlayerZombies - currentSpecials;
    if (g_iInfectedBotQueue > availableSlots)
    {
        if (slot >= 1 && slot <= MaxClients) g_bSpawnReady[slot] = true;
        return Plugin_Stop;
    }

    if (g_bDisableSpawnsTank)
    {
        for (int i = 1; i <= MaxClients; i++)
        {
            if (!IsClientInGame(i)) continue;
            if (GetClientTeam(i) == TEAM_INFECTED && IsPlayerTank(i) && IsPlayerAlive(i))
            {
                if (slot >= 1 && slot <= MaxClients) g_bSpawnReady[slot] = true;
                return Plugin_Stop;
            }
        }
    }

    static float s_fLastHeavyFrame  = -1.0;
    static float s_fLastHeavySearch = -100.0;
    float fFrame = GetGameTime();

    if (fFrame == s_fLastHeavyFrame || (fFrame - s_fLastHeavySearch) < HEAVY_SEARCH_GAP)
    {
        ScheduleSpawnRaw(0.2 + GetRandomFloat(0.0, 0.1));
        return Plugin_Stop;
    }
    s_fLastHeavyFrame  = fFrame;
    s_fLastHeavySearch = fFrame;

    int traceStartTick = GetSysTickCount();

    int targetSurvivor = -1;
    float targetPos[3];
    bool found = false;

    if (GetRandomAliveSurvivor(targetSurvivor))
        found = GetRandomSpawnPosition(targetSurvivor, targetPos, traceStartTick);
    else
    {
        static float s_fNoSurvLog = 0.0;
        float fNow = GetGameTime();
        if (fNow - s_fNoSurvLog > 5.0)
        {
            LogMessage("[IB] 无可用幸存者，稍后重试");
            s_fNoSurvLog = fNow;
        }
    }

    if (!found)
    {
        ScheduleRetry();
        return Plugin_Stop;
    }

    int botType = BotTypeNeeded();
    int zc = 0;

    switch (botType)
    {
        case 1: zc = ZC_HUNTER;
        case 2: zc = ZC_SMOKER;
        case 3: zc = ZC_BOOMER;
        case 4: zc = ZC_SPITTER;
        case 5: zc = ZC_JOCKEY;
        case 6: zc = ZC_CHARGER;
        case 7: zc = g_iZombieClassTank;
    }

    if (zc == 0)
    {
        if (slot >= 1 && slot <= MaxClients) g_bSpawnReady[slot] = true;
        return Plugin_Stop;
    }

    if (zc >= 1 && zc <= 6 && !CanSpawnClass(zc))
    {
        if (slot >= 1 && slot <= MaxClients) g_bSpawnReady[slot] = true;
        return Plugin_Stop;
    }

    ReserveClass(zc);
    int spawnedClient = L4D2_SpawnSpecial(zc, targetPos, NULL_VECTOR);
    ReleaseClass(zc);

    if (spawnedClient <= 0)
    {
        LogMessage("[IB] L4D2_SpawnSpecial failed (slot=%d), retrying", slot);
        ScheduleRetry();
        return Plugin_Stop;
    }

    int actualClass = GetEntProp(spawnedClient, Prop_Send, "m_zombieClass");
    if (actualClass != zc)
    {
        LogMessage("[IB] CLASS MISMATCH: req=%d actual=%d client=%d, recycling", zc, actualClass, spawnedClient);

        if (zc >= 1 && zc <= 6)
            SetEntProp(spawnedClient, Prop_Send, "m_zombieClass", zc);

        CreateTimer(0.1, Timer_KickBot, spawnedClient, TIMER_FLAG_NO_MAPCHANGE);
        ScheduleRetry();
        return Plugin_Stop;
    }

    if (zc >= 1 && zc <= 6)
        g_fClassCooldown[zc] = GetGameTime() + CLASS_COOLDOWN;

    SetEntityMoveType(spawnedClient, MOVETYPE_WALK);

    float actualPos[3];
    GetClientAbsOrigin(spawnedClient, actualPos);
    float snapDist = GetVectorDistance(actualPos, targetPos);

    if (snapDist > 50.0)
    {
        TeleportEntity(spawnedClient, targetPos, NULL_VECTOR, NULL_VECTOR);
        GetClientAbsOrigin(spawnedClient, actualPos);
        snapDist = GetVectorDistance(actualPos, targetPos);

        if (snapDist > 50.0)
        {
            LogMessage("[IB] Force teleport failed (dist=%.0f), recycling", snapDist);
            CreateTimer(0.1, Timer_KickBot, spawnedClient, TIMER_FLAG_NO_MAPCHANGE);
            ScheduleRetry();
            return Plugin_Stop;
        }
    }

    if (snapDist > 0.1)
    {
        float verifyCheckPos[3];
        verifyCheckPos = actualPos;
        int verifyStartTick = GetSysTickCount();

        if (!g_bLastSpawnIsFallback
            && IsPosVisibleToAnySurvivor(verifyCheckPos, verifyStartTick, SPAWN_MAX_TRACE_TIME_MS))
        {
            CreateTimer(0.1, Timer_KickBot, spawnedClient, TIMER_FLAG_NO_MAPCHANGE);
            ScheduleRetry();
            return Plugin_Stop;
        }

        if (!g_bLastSpawnIsFallback
            && IsPredictedLandingVisible(verifyCheckPos, verifyStartTick))
        {
            CreateTimer(0.1, Timer_KickBot, spawnedClient, TIMER_FLAG_NO_MAPCHANGE);
            ScheduleRetry();
            return Plugin_Stop;
        }

        float checkSafeDist = g_bLastSpawnIsFallback ? SPAWN_FALLBACK_SAFE_DIST : g_hCvarSpawnMinRange.FloatValue;
        if (!IsSafeDistanceToAllSurvivors(verifyCheckPos, checkSafeDist))
        {
            CreateTimer(0.1, Timer_KickBot, spawnedClient, TIMER_FLAG_NO_MAPCHANGE);
            ScheduleRetry();
            return Plugin_Stop;
        }

        if (IsSpawnPointInHurtTrigger(verifyCheckPos))
        {
            CreateTimer(0.1, Timer_KickBot, spawnedClient, TIMER_FLAG_NO_MAPCHANGE);
            ScheduleRetry();
            return Plugin_Stop;
        }
    }

    g_iSpawnFailStreak = 0;

    static float s_fSpawnedLog = 0.0;
    float fNow2 = GetGameTime();
    if (fNow2 - s_fSpawnedLog > 1.0)
    {
        LogMessage("[IB] Spawned: class=%d client=%d dist=%.0f fallback=%d",
            zc, spawnedClient, snapDist, g_bLastSpawnIsFallback);
        s_fSpawnedLog = fNow2;
    }

    AcceptSpawnAndScheduleNext(spawnedClient);
    return Plugin_Stop;
}

public Action Timer_SpawnInfectedBotDirector(Handle timer, any botNeeded)
{
    bool resetGhost[MAXPLAYERS + 1];

    for (int i = 1; i <= MaxClients; i++)
    {
        if (IsClientInGame(i) && !IsFakeClient(i) && GetClientTeam(i) == TEAM_INFECTED)
        {
            if (IsPlayerGhost(i)) { resetGhost[i] = true; SetGhostStatus(i, false); }
        }
    }

    int anyClient = GetAnyClient();
    bool temp = false;

    if (anyClient == -1)
    {
        anyClient = CreateFakeClient("Bot");
        if (!anyClient) { LogError("[IB] 临时Bot创建失败"); return Plugin_Stop; }
        temp = true;
    }

    g_bSpecialHalt = true;
    switch (botNeeded)
    {
        case 1: CheatCommand(anyClient, g_sSpawnCmd, "smoker auto");
        case 2: CheatCommand(anyClient, g_sSpawnCmd, "boomer auto");
        case 3: CheatCommand(anyClient, g_sSpawnCmd, "hunter auto");
        case 4: CheatCommand(anyClient, g_sSpawnCmd, "spitter auto");
        case 5: CheatCommand(anyClient, g_sSpawnCmd, "jockey auto");
        case 6: CheatCommand(anyClient, g_sSpawnCmd, "charger auto");
    }
    g_bSpecialHalt = false;

    for (int i = 1; i <= MaxClients; i++)
        if (resetGhost[i]) SetGhostStatus(i, true);

    if (temp) CreateTimer(0.1, Timer_KickBot, anyClient, TIMER_FLAG_NO_MAPCHANGE);
    return Plugin_Stop;
}

// ===== 类型选择 =====
int BotTypeNeeded()
{
    ExpireClassReservations();

    int tanks = 0;
    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (GetClientTeam(i) != TEAM_INFECTED || !PlayerIsAlive(i)) continue;
        if (IsPlayerTank(i)) tanks++;
    }

    int availableTypes[7];
    int count = 0;

    if (CanSpawnClass(ZC_HUNTER)  && g_bCanSpawnHunter)  availableTypes[count++] = 1;
    if (CanSpawnClass(ZC_SMOKER)  && g_bCanSpawnSmoker)  availableTypes[count++] = 2;
    if (CanSpawnClass(ZC_BOOMER)  && g_bCanSpawnBoomer)  availableTypes[count++] = 3;
    if (g_bIsL4D2)
    {
        if (CanSpawnClass(ZC_SPITTER) && g_bCanSpawnSpitter) availableTypes[count++] = 4;
        if (CanSpawnClass(ZC_JOCKEY)  && g_bCanSpawnJockey)  availableTypes[count++] = 5;
        if (CanSpawnClass(ZC_CHARGER) && g_bCanSpawnCharger) availableTypes[count++] = 6;
    }
    if (tanks < g_iTankLimit) availableTypes[count++] = 7;

    if (count == 0) return 0;
    return availableTypes[GetURandomIntRange(0, count - 1)];
}

// ===== 定时器 =====
public Action Timer_MaxSpecialsSet(Handle timer)
{
    g_hZMaxPlayerZombies.SetInt(g_iMaxPlayerZombies);
    return Plugin_Stop;
}

public Action Timer_PlayerLeftStart(Handle timer)
{
    if (LeftStartArea())
    {
        if (!g_bLeftSaveRoom)
        {
            char sGameName[64];
            g_hCvarGameMode.GetString(sGameName, sizeof(sGameName));

            if (StrEqual(sGameName, "mutation15", false))
            {
                SetCvarInt("survival_max_smokers", 0);
                SetCvarInt("survival_max_boomers", 0);
                SetCvarInt("survival_max_hunters", 0);
                SetCvarInt("survival_max_jockeys", 0);
                SetCvarInt("survival_max_spitters", 0);
                SetCvarInt("survival_max_chargers", 0);
                return Plugin_Stop;
            }

            g_bLeftSaveRoom      = true;
            g_bHasRoundStarted   = true;
            g_fLastSpawnSuccess  = GetGameTime();
            g_fLastWatchdogReset = 0.0;

            EnableAllSpawnFlags();

            g_bInitialSpawn    = true;
            g_bFirstWaveArmed  = true;
            g_bFirstWaveLogged = false;

            StartIdleCheck();

            CheckIfBotsNeeded(false, true);
            g_bFirstWaveArmed = false;

            CountInfected();
            LogMessage("[IB] First wave armed: survivors=%d max=%d queued=%d delay=%ds window=%ds",
                TotalSurvivors_(), g_iMaxPlayerZombies, g_iInfectedBotQueue,
                g_iInitialSpawnInt, g_iInitialSpawnInt + 10);

            CreateTimer(float(g_iInitialSpawnInt + 10), Timer_InitialSpawnReset, _, TIMER_FLAG_NO_MAPCHANGE);

            if (g_hPeriodicCheckTimer != null) delete g_hPeriodicCheckTimer;
            g_hPeriodicCheckTimer = CreateTimer(1.0, Timer_PeriodicCheck, _, TIMER_REPEAT | TIMER_FLAG_NO_MAPCHANGE);
        }
    }
    else CreateTimer(1.0, Timer_PlayerLeftStart, _, TIMER_FLAG_NO_MAPCHANGE);

    return Plugin_Stop;
}

public Action Timer_InitialSpawnReset(Handle timer)
{
    if (g_bInitialSpawn)
    {
        g_bInitialSpawn   = false;
        g_bFirstWaveArmed = false;
        LogMessage("[IB] First wave window closed, switching to normal spawn interval");
    }
    return Plugin_Stop;
}

public Action Timer_CheckIfBotsNeededLater(Handle timer, any spawnImmediately)
{
    CheckIfBotsNeeded(spawnImmediately != 0, false);
    return Plugin_Stop;
}

public Action Timer_PeriodicCheck(Handle timer)
{
    if (g_bHasRoundEnded || !g_bLeftSaveRoom) return Plugin_Continue;
    if (g_bDirectorSpawn) return Plugin_Continue;

    ValidateIncapCount();
    ValidateTankBonus();
    ExpireClassReservations();

    int realTimers = 0;
    for (int i = 1; i <= MaxClients; i++)
        if (g_hSpawnTimer[i] != null) realTimers++;

    if (g_iInfectedBotQueue != realTimers)
    {
        static float s_fReconcileLog = 0.0;
        float fNow = GetGameTime();
        if (fNow - s_fReconcileLog > 3.0)
        {
            LogMessage("[IB] Queue reconcile: %d -> %d", g_iInfectedBotQueue, realTimers);
            s_fReconcileLog = fNow;
        }
        g_iInfectedBotQueue = realTimers;
    }

    CheckIfBotsNeeded(false, false);

    float now = GetGameTime();

    if (!g_bHasRoundStarted) g_bHasRoundStarted = true;
    if (now - g_fLastWatchdogReset < 5.0) return Plugin_Continue;

    CountInfected();
    int alive   = g_iInfectedBotCount + g_iInfectedRealCount;
    int deficit = g_iMaxPlayerZombies - alive;
    if (deficit <= 0) return Plugin_Continue;

    int pending = 0;
    for (int i = 1; i <= MaxClients; i++)
        if (g_hSpawnTimer[i] != null || g_bSpawnReady[i]) pending++;

    bool structural = (pending == 0) || (g_iInfectedBotQueue > g_iMaxPlayerZombies);

    int surv = TrueNumberOfSurvivors();
    if (surv < 1) surv = 1;
    float maxWait = g_fInfectedSpawnTimeMax
                  + g_hCvarIncapSpawnDelay.FloatValue * float(surv)
                  + 15.0;
    if (maxWait < 25.0) maxWait = 25.0;

    bool timeout = (!g_bInitialSpawn && (now - g_fLastSpawnSuccess > maxWait));

    if (structural || timeout)
    {
        LogMessage("[IB] Watchdog(%s): alive=%d max=%d queue=%d pending=%d thr=%.0fs",
            structural ? "struct" : "timeout",
            alive, g_iMaxPlayerZombies, g_iInfectedBotQueue, pending, maxWait);

        g_fLastWatchdogReset = now;
        ClearAllSpawnTimers();
        g_fLastSpawnSuccess = now;
        CheckIfBotsNeeded(true, false);
    }
    return Plugin_Continue;
}

public Action Timer_InfectedBotBooterVersus(Handle timer)
{
    if (g_iGameMode != GAMEMODE_VERSUS || g_bIsL4D2) return Plugin_Stop;

    int total;
    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (GetClientTeam(i) != TEAM_INFECTED) continue;
        if (!IsPlayerTank(i) || (IsPlayerTank(i) && !PlayerIsAlive(i))) total++;
    }

    if (total + g_iInfectedBotQueue > g_iMaxPlayerZombies)
    {
        int kick = total + g_iInfectedBotQueue - g_iMaxPlayerZombies;
        int kicked = 0;

        for (int i = 1; i <= MaxClients && kicked < kick; i++)
        {
            if (!IsClientInGame(i) || !IsFakeClient(i)) continue;
            if (GetClientTeam(i) != TEAM_INFECTED) continue;
            if (!IsPlayerTank(i) || (IsPlayerTank(i) && !PlayerIsAlive(i)))
            {
                CreateTimer(0.1, Timer_KickBot, i, TIMER_FLAG_NO_MAPCHANGE);
                kicked++;
            }
        }
    }
    return Plugin_Stop;
}

public Action Timer_SetUpBotGhost(Handle timer, any client)
{
    if (IsClientInGame(client))
    {
        if (!g_bAlreadyGhostedBot[client])
        {
            SetGhostStatus(client, true);
            SetEntityMoveType(client, MOVETYPE_NONE);
            CreateTimer(g_hCvarBotGhostTime.FloatValue, Timer_RestoreBotGhost, client, TIMER_FLAG_NO_MAPCHANGE);
        }
        else g_bAlreadyGhostedBot[client] = false;
    }
    return Plugin_Stop;
}

public Action Timer_RestoreBotGhost(Handle timer, any client)
{
    if (IsClientInGame(client))
    {
        SetGhostStatus(client, false);
        SetEntityMoveType(client, MOVETYPE_WALK);
    }
    return Plugin_Stop;
}

// 【重要】只能 ForcePlayerSuicide，绝不能 KickClient！
public Action Timer_KickBot(Handle timer, any client)
{
    if (client < 1 || client > MaxClients) return Plugin_Stop;

    if (IsClientInGame(client) && !IsClientInKickQueue(client) && IsFakeClient(client))
        ForcePlayerSuicide(client);

    return Plugin_Stop;
}

// ===== 诊断命令 =====
public Action Cmd_IBDebug(int client, int args)
{
    int timers = 0, ready = 0, ghost = 0, dead = 0;

    for (int i = 1; i <= MaxClients; i++)
    {
        if (g_hSpawnTimer[i] != null) timers++;
        if (g_bSpawnReady[i]) ready++;

        if (IsClientInGame(i) && IsFakeClient(i) && GetClientTeam(i) == TEAM_INFECTED)
        {
            if (!PlayerIsAlive(i)) dead++;
            else if (IsPlayerGhost(i)) ghost++;
        }
    }
    CountInfected();

    int surv = TrueNumberOfSurvivors();
    if (surv < 1) surv = 1;
    float maxWait = g_fInfectedSpawnTimeMax + g_hCvarIncapSpawnDelay.FloatValue * float(surv) + 15.0;
    if (maxWait < 25.0) maxWait = 25.0;

    ReplyToCommand(client, "[IB] v%s started=%d ended=%d left=%d | max=%d bot=%d real=%d queue=%d timers=%d ready=%d ghost=%d dead=%d slots=%d/%d",
        PLUGIN_VERSION, g_bHasRoundStarted, g_bHasRoundEnded, g_bLeftSaveRoom,
        g_iMaxPlayerZombies, g_iInfectedBotCount, g_iInfectedRealCount, g_iInfectedBotQueue,
        timers, ready, ghost, dead, GetClientCount(true), MaxClients);

    ReplyToCommand(client, "[IB] firstwave=%d armed=%d director=%d | interval=%.1f~%.1f cap=%.1f watchdog=%.0fs lastOK=%.1fs ago",
        g_bInitialSpawn, g_bFirstWaveArmed, g_bDirectorSpawn,
        g_fInfectedSpawnTimeMin, g_fInfectedSpawnTimeMax, GetSpawnDelayCap(), maxWait,
        GetGameTime() - g_fLastSpawnSuccess);

    ReplyToCommand(client, "[IB] 360scan near=%.0f~%.0f visMin=%.0f visMax=%.0f(+%.0f) farExt=%.0f | dir=%d extraDir=%d extraMs=%d path=%d/%d/%d gap=%.2fs streak=%d",
        g_hCvarSpawnMinRange.FloatValue, g_hCvarSpawnMaxRange.FloatValue,
        g_hCvarSpawnVisMinRange.FloatValue,
        g_hCvarSpawnMaxRange.FloatValue + VIS_SCAN_EXTEND, VIS_SCAN_EXTEND,
        SCAN_FAR_EXT, RoundToNearest(360.0 / SCAN_ANGLE_STEP), VIS_SCAN_DIRS,
        VIS_SCAN_EXTRA_MS, SCAN_PATH_NEAR, SCAN_PATH_VIS, SCAN_PATH_FAR,
        HEAVY_SEARCH_GAP, g_iSpawnFailStreak);

    ReplyToCommand(client, "[IB] flags H=%d S=%d B=%d Sp=%d J=%d C=%d | periodic=%d idle=%d incap=%d tank=%d",
        g_bCanSpawnHunter, g_bCanSpawnSmoker, g_bCanSpawnBoomer,
        g_bCanSpawnSpitter, g_bCanSpawnJockey, g_bCanSpawnCharger,
        g_hPeriodicCheckTimer != null, g_hIdleCheckTimer != null, g_iCurrentIncapCount, g_bTankBonusActive);

    ReplyToCommand(client, "[IB] reserved=[%d,%d,%d,%d,%d,%d] cd=[%.1f,%.1f,%.1f,%.1f,%.1f,%.1f]",
        g_iClassReserved[1], g_iClassReserved[2], g_iClassReserved[3],
        g_iClassReserved[4], g_iClassReserved[5], g_iClassReserved[6],
        g_fClassCooldown[1] - GetGameTime(), g_fClassCooldown[2] - GetGameTime(),
        g_fClassCooldown[3] - GetGameTime(), g_fClassCooldown[4] - GetGameTime(),
        g_fClassCooldown[5] - GetGameTime(), g_fClassCooldown[6] - GetGameTime());

    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i) || GetClientTeam(i) != TEAM_SURVIVORS) continue;

        if (g_bIsL4D2)
        {
            ReplyToCommand(client, "[IB] surv %N alive=%d incap=%d ledge=%d/%d tongue=%d pounce=%d jockey=%d pummel=%d carry=%d",
                i, PlayerIsAlive(i),
                GetEntProp(i, Prop_Send, "m_isIncapacitated"),
                GetEntProp(i, Prop_Send, "m_isHangingFromLedge"),
                GetEntProp(i, Prop_Send, "m_isFallingFromLedge"),
                GetEntPropEnt(i, Prop_Send, "m_tongueOwner"),
                GetEntPropEnt(i, Prop_Send, "m_pounceAttacker"),
                GetEntPropEnt(i, Prop_Send, "m_jockeyAttacker"),
                GetEntPropEnt(i, Prop_Send, "m_pummelAttacker"),
                GetEntPropEnt(i, Prop_Send, "m_carryAttacker"));
        }
        else
        {
            ReplyToCommand(client, "[IB] surv %N alive=%d incap=%d ledge=%d tongue=%d pounce=%d",
                i, PlayerIsAlive(i),
                GetEntProp(i, Prop_Send, "m_isIncapacitated"),
                GetEntProp(i, Prop_Send, "m_isHangingFromLedge"),
                GetEntPropEnt(i, Prop_Send, "m_tongueOwner"),
                GetEntPropEnt(i, Prop_Send, "m_pounceAttacker"));
        }
    }

    int t;
    ReplyToCommand(client, "[IB] GetRandomAliveSurvivor = %d", GetRandomAliveSurvivor(t));
    return Plugin_Handled;
}

public Action Cmd_IBReset(int client, int args)
{
    ClearAllSpawnTimers();
    ClearClassCooldown();
    g_fLastSpawnSuccess  = GetGameTime();
    g_fLastWatchdogReset = 0.0;

    if (!g_bHasRoundEnded && g_bLeftSaveRoom) CheckIfBotsNeeded(true, false);

    ReplyToCommand(client, "[IB] 调度池已强制重置");
    return Plugin_Handled;
}

// ===== Client 连接 =====
public void OnClientConnected(int client)
{
    if (IsFakeClient(client)) return;

    g_iPlayersInServer++;

    if (g_iPlayersInServer == 1 && g_bOldData)
    {
        g_iOnlyClient = GetClientUserId(client);
        g_bOldData = false;
        g_bHasRoundEnded = false;
        g_iInfectedBotQueue = 0;
    }
}

public void OnClientDisconnect(int client)
{
    int wasTeam = 0;
    if (IsClientInGame(client)) wasTeam = GetClientTeam(client);
    else if (client > 0 && client <= MaxClients && g_bIsIncap[client]) wasTeam = TEAM_SURVIVORS;

    if (client > 0 && client <= MaxClients && g_bIsIncap[client])
    {
        g_bIsIncap[client] = false;
        if (g_iCurrentIncapCount > 0) g_iCurrentIncapCount--;
        if (wasTeam != TEAM_SURVIVORS) wasTeam = TEAM_SURVIVORS;
    }

    if (client > 0 && client <= MaxClients)
    {
        g_bAlreadyGhosted[client] = false;
        g_bAlreadyGhostedBot[client] = false;
        g_bPlayerHasEnteredStart[client] = false;
        g_fIdleTime[client] = 0.0;
        g_fLastPos[client][0] = 0.0;
        g_fLastPos[client][1] = 0.0;
        g_fLastPos[client][2] = 0.0;
        g_fSpawnLifeTime[client] = 0.0;
        g_bPendingLifespanKill[client] = false;
        g_fBotFirstSeen[client] = 0.0;
        g_fGhostSince[client] = 0.0;
    }

    if (!IsFakeClient(client)) g_iPlayersInServer--;

    if (g_iPlayersInServer <= 0 && !IsFakeClient(client))
    {
        g_iPlayersInServer = 0;
        g_bLeftSaveRoom    = false;
        g_bHasRoundEnded   = true;
        g_bHasRoundStarted = false;
        g_bInitialSpawn    = false;
        g_bFirstWaveArmed  = false;
        g_bDirectorCvarsModified = false;
        g_bOldData = true;
        g_iLastSurvivors = -1;
        g_iCurrentIncapCount = 0;
        g_bTankBonusActive = false;
        g_fLastWatchdogReset = 0.0;

        delete g_hDebouncePrintTimer;
        g_hDebouncePrintTimer = null;
        ClearAllSpawnTimers();
        ClearClassCooldown();
        StopIdleCheck();

        if (g_hPeriodicCheckTimer != null) { delete g_hPeriodicCheckTimer; g_hPeriodicCheckTimer = null; }

        for (int i = 1; i <= MaxClients; i++)
        {
            g_bAlreadyGhosted[i] = false;
            g_bPlayerHasEnteredStart[i] = false;
            g_bIsIncap[i] = false;
            g_fSpawnLifeTime[i] = 0.0;
            g_bPendingLifespanKill[i] = false;
            g_fBotFirstSeen[i] = 0.0;
            g_fGhostSince[i] = 0.0;
        }
    }
    else if (wasTeam == TEAM_SURVIVORS && g_bLeftSaveRoom)
        RequestFrame(DelayedRecalculate);
}

public void OnMapEnd()
{
    g_bHasRoundStarted = false;
    g_bHasRoundEnded   = true;
    g_bLeftSaveRoom    = false;
    g_bInitialSpawn    = false;
    g_bFirstWaveArmed  = false;
    g_bFirstWaveLogged = false;
    g_iLastSurvivors   = -1;
    g_iCurrentIncapCount = 0;
    g_bTankBonusActive = false;
    g_fLastSpawnSuccess  = 0.0;
    g_fLastWatchdogReset = 0.0;

    ClearClassCooldown();

    delete g_hDebouncePrintTimer;
    g_hDebouncePrintTimer = null;
    ClearAllSpawnTimers();
    StopIdleCheck();

    if (g_hPeriodicCheckTimer != null) { delete g_hPeriodicCheckTimer; g_hPeriodicCheckTimer = null; }
    if (g_hHurtTriggers != null) { delete g_hHurtTriggers; g_hHurtTriggers = null; }

    g_fHurtRefreshTime = 0.0;
    g_iSpawnSepCount = 0;
    g_iSpawnSepHead  = 0;

    for (int i = 1; i <= MaxClients; i++)
    {
        g_fIdleTime[i] = 0.0;
        g_fLastPos[i][0] = 0.0;
        g_fLastPos[i][1] = 0.0;
        g_fLastPos[i][2] = 0.0;
        g_fSpawnLifeTime[i] = 0.0;
        g_bPendingLifespanKill[i] = false;
        g_bIsIncap[i] = false;
        g_fBotFirstSeen[i] = 0.0;
        g_fGhostSince[i] = 0.0;
    }
}

// ===== 计数 =====
void CountInfected()
{
    g_iInfectedBotCount  = 0;
    g_iInfectedRealCount = 0;

    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientInGame(i)) continue;
        if (GetClientTeam(i) != TEAM_INFECTED) continue;
        if (IsPlayerTank(i)) continue;
        if (!PlayerIsAlive(i)) continue;

        if (IsFakeClient(i)) g_iInfectedBotCount++;
        else g_iInfectedRealCount++;
    }
}

int TrueNumberOfSurvivors()
{
    int total;
    for (int i = 1; i <= MaxClients; i++)
        if (IsClientInGame(i) && GetClientTeam(i) == TEAM_SURVIVORS) total++;
    return total;
}

int TotalSurvivors_()
{
    int total;
    for (int i = 1; i <= MaxClients; i++)
    {
        if (!IsClientConnected(i)) continue;
        if (!IsClientInGame(i)) continue;
        if (IsClientInKickQueue(i)) continue;
        if (GetClientTeam(i) != TEAM_SURVIVORS) continue;
        if (!PlayerIsAlive(i)) continue;
        total++;
    }
    return total;
}

int HumansOnInfected()
{
    int total;
    for (int i = 1; i <= MaxClients; i++)
        if (IsClientInGame(i) && GetClientTeam(i) == TEAM_INFECTED && !IsFakeClient(i)) total++;
    return total;
}

// ===== 玩家状态 =====
bool IsPlayerGhost(int c)   { return view_as<bool>(GetEntProp(c, Prop_Send, "m_isGhost")); }
bool PlayerIsAlive(int c)   { return !view_as<bool>(GetEntProp(c, Prop_Send, "m_lifeState")); }
bool IsPlayerSmoker(int c)  { return GetEntProp(c, Prop_Send, "m_zombieClass") == ZC_SMOKER; }
bool IsPlayerBoomer(int c)  { return GetEntProp(c, Prop_Send, "m_zombieClass") == ZC_BOOMER; }
bool IsPlayerHunter(int c)  { return GetEntProp(c, Prop_Send, "m_zombieClass") == ZC_HUNTER; }
bool IsPlayerSpitter(int c) { return GetEntProp(c, Prop_Send, "m_zombieClass") == ZC_SPITTER; }
bool IsPlayerJockey(int c)  { return GetEntProp(c, Prop_Send, "m_zombieClass") == ZC_JOCKEY; }
bool IsPlayerCharger(int c) { return GetEntProp(c, Prop_Send, "m_zombieClass") == ZC_CHARGER; }
bool IsPlayerTank(int c)    { return GetEntProp(c, Prop_Send, "m_zombieClass") == g_iZombieClassTank; }
void SetGhostStatus(int c, bool g) { SetEntProp(c, Prop_Send, "m_isGhost", g ? 1 : 0); }

// ===== 工具 =====
bool LeftStartArea()
{
    int ent = GetPlayerResourceEntity();
    if (ent > 0 && IsValidEntity(ent))
        return view_as<bool>(GetEntProp(ent, Prop_Send, "m_hasAnySurvivorLeftSafeArea"));

    int maxEnts = GetMaxEntities();
    for (int i = MaxClients + 1; i <= maxEnts; i++)
    {
        if (!IsValidEntity(i)) continue;
        char netClass[64];
        GetEntityNetClass(i, netClass, sizeof(netClass));
        if (StrEqual(netClass, "CTerrorPlayerResource"))
            return view_as<bool>(GetEntProp(i, Prop_Send, "m_hasAnySurvivorLeftSafeArea"));
    }
    return false;
}

int GetAnyClient()
{
    for (int i = 1; i <= MaxClients; i++)
        if (IsClientInGame(i)) return i;
    return -1;
}

int GetURandomIntRange(int min, int max)
{
    if (min > max) { int tmp = min; min = max; max = tmp; }
    int span = max - min + 1;
    if (span <= 0) return min;
    return (GetURandomInt() % span) + min;
}

float GetURandomFloatRange(float min, float max)
{
    if (min > max) { float tmp = min; min = max; max = tmp; }
    if (max - min < 0.05) return min;
    return min + GetURandomFloat() * (max - min);
}

void CheatCommand(int client, const char[] command, const char[] arguments = "")
{
    int userFlags = GetUserFlagBits(client);
    SetUserFlagBits(client, ADMFLAG_ROOT);

    int flags = GetCommandFlags(command);
    SetCommandFlags(command, flags & ~FCVAR_CHEAT);

    FakeClientCommand(client, "%s %s", command, arguments);

    SetCommandFlags(command, flags);
    if (IsClientInGame(client)) SetUserFlagBits(client, userFlags);
}
