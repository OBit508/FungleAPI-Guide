# BasePlugin
using BepInEx;
using BepInEx.Unity.IL2CPP;
using FungleAPI;
using FungleAPI.PluginLoading;
using TMPro;
using System.Collections;

[BepInProcess("Among Us.exe")]
[BepInPlugin("your_mod_id", "your_mod_name", "your_mod_version")]
[BepInDependency(FungleAPIPlugin.ModId)]
public class BaseModPlugin : BasePlugin, IFungleBasePlugin
{
    public static BaseModPlugin Instance => FunglePlugin<BaseModPlugin>.Instance; // Optional
    public static ModPlugin Plugin => FunglePlugin<BaseModPlugin>.Plugin; // You will use this Plugin every time
    public string ModName => "your_mod_name"; // Put here your mod name
    public string ModVersion => "your_mod_version"; // Put here your mod version
    public override void Load() // Load your mod
    {
    }
    public void LoadAssets() // Use this to load your assets
    {
    }
    public void OnRegisterInFungleAPI() // Called when your mod got registered
    {
    }
    public IEnumerator CoLoadOnMainScreen(TextMeshPro loadingText) // Use this to load something big
    {
        yield return null;
    }
}
// The BasePlugin is the main loader from your mod, you need to have one
# Events
// Events are a simple way to trigger some points of your code when a certain action happens
// You can easily create a event with
using FungleAPI.Event.Types;

public class ModEvent : FungleEvent
{
}
// When created you can register some static method to be called when you trigger this event with
[FungleAPI.Event.EventRegister]
public static void EventMethod(ModEvent modEvent)
{
}
// To trigger your event you need to use the EventManager like this
FungleAPI.Event.EventManager.CallEvent(new ModEvent());
// The API have some ready-made events that you can use like
public class OnCloseDoor : FungleEvent
{
    public SystemTypes Room { get; internal set; }
}
public class OnCompleteTask : FungleEvent
{
    public PlayerControl Player { get; internal set; }
    public PlayerTask Task { get; internal set; }
}
public class OnEject : FungleEvent
{
    public ExileController Controller { get; internal set; }
    public NetworkedPlayerInfo Target { get; internal set; }
}
public class OnEndMeeting : FungleEvent
{
    public MeetingHud Meeting { get; internal set; }
}
public class OnEnterVent : FungleEvent
{
    public PlayerControl Player { get; internal set; }
    public Vent Vent { get; internal set; }
}
// You can see the rest of the events on FungleAPI.Event.Types
# GameOvers
// GameOver is a way to end the game with a reason and winners
// You can easily create a GameOver with
using System.Collections.Generic;
using FungleAPI.GameOver;
using Hazel;
using InnerNet;
using UnityEngine;

public class ModGameOver : CustomGameOver
{
    public override GameOverReason Reason { get; } = GameOverManager.GetValidGameOver(); // You need this to avoid errors
    public override string WinText => "win_text"; // The text that will apper in GameOver screen
    public override Color BackgroundColor => Color.cyan; // The GameOver screen background color
    public override Color NameColor => Color.white; // The color of players names
    public override AudioClip Clip => null; // The sound that will play when the GameOver screen apper, if you let null will play the vanilla one
    public override List<NetworkedPlayerInfo> GetWinners() // Use this to say for the API who will win
    {
        List<NetworkedPlayerInfo> winners = new List<NetworkedPlayerInfo>();
        foreach (NetworkedPlayerInfo networkedPlayerInfo in GameData.Instance.AllPlayers)
        {
            if (networkedPlayerInfo.Role.DidWin(Reason))
            {
                winners.Add(networkedPlayerInfo);
            }
        }
        return winners;
    }
    public override void Serialize(MessageWriter messageWriter) // Use this to send the winners to others clients
    {
        Winners.Clear();
        List<NetworkedPlayerInfo> winners = GetWinners();
        messageWriter.Write(winners.Count);
        foreach (NetworkedPlayerInfo winner in winners)
        {
            Winners.Add(new CachedPlayerData(winner));
            messageWriter.WriteNetObject(winner);
        }
    }
    public override void Deserialize(MessageReader messageReader) // Use this to read the winners
    {
        Winners.Clear();
        int count = messageReader.ReadInt32();
        for (int i = 0; i < count; i++)
        {
            Winners.Add(new CachedPlayerData(messageReader.ReadNetObject<NetworkedPlayerInfo>()));
        }
    }
    public override void OnSetEverythingUp(EndGameManager endGameManager) // Optional
    {
    }
}
// You can get the GameOver Instance with
FungleAPI.GameOver.GameOverManager.Instance<ModGameOver>();
// To call a GameOver you will need to use the GameOverManager using one of this methods
FungleAPI.GameOver.GameOverManager.RpcEndGame<ModGameOver>(GameManager.Instance);
FungleAPI.GameOver.GameOverManager.RpcEndGame(GameManager.Instance, FungleAPI.GameOver.GameOverManager.Instance<ModGameOver>());
FungleAPI.GameOver.GameOverManager.RpcEndGame(GameManager.Instance, ModdedTeam); // ModdedTeam is something that still will be explained
FungleAPI.GameOver.GameOverManager.RpcEndGame(GameManager.Instance, new List<NetworkedPlayerInfo>(), "win_text", Color.cyan, Color.white); // Custom way to call a GameOver without creating a GameOver class
// You will able to use one of the Vanilla ones GameOvers like
FungleAPI.GameOver.Ends.CrewmateDisconnect
FungleAPI.GameOver.Ends.CrewmatesByTask
FungleAPI.GameOver.Ends.CrewmatesByVote
FungleAPI.GameOver.Ends.HideAndSeek_CrewmatesByTimer
FungleAPI.GameOver.Ends.HideAndSeek_ImpostorsByKills
FungleAPI.GameOver.Ends.ImpostorDisconnect
FungleAPI.GameOver.Ends.ImpostorsByKill
FungleAPI.GameOver.Ends.ImpostorsBySabotage,
FungleAPI.GameOver.Ends.ImpostorsByVote
FungleAPI.GameOver.Ends.NeutralGameOver
# RPCs
// Rpc is a message from a client to another client, normally used to sync events of the game
// Here are all the 4 Rpc Base class, use this to create one with the base class
using FungleAPI.Attributes;
using FungleAPI.Networking;
using Hazel;
using InnerNet;

namespace FungleAPI.Base.Rpc
{
    [FungleIgnore]
    public class SimpleRpc : BaseRpcHelper<InnerNetObject>
    {
        public void Send(InnerNetObject innerNetObject, SendOption sendOption = SendOption.Reliable, int targetClientId = -1)
        {
            CustomRpcManager.SendRpc(innerNetObject, delegate (MessageWriter writer)
            {
                writer.WriteRPC(this);
                Write(innerNetObject, writer);
            }, sendOption, targetClientId);
        }
        public virtual void Write(MessageWriter messageWriter)
        {
        }
        public virtual void Write(InnerNetObject innerNetObject, MessageWriter messageWriter)
        {
            Write(messageWriter);
        }
    }
    [FungleIgnore]
    public class SimpleRpc<TNetObject> : BaseRpcHelper<TNetObject> where TNetObject : InnerNetObject
    {
        public void Send(TNetObject innerNetObject, SendOption sendOption = SendOption.Reliable, int targetClientId = -1)
        {
            CustomRpcManager.SendRpc(innerNetObject, delegate (MessageWriter writer)
            {
                writer.WriteRPC(this);
                Write(innerNetObject, writer);
            }, sendOption, targetClientId);
        }
        public virtual void Write(MessageWriter messageWriter)
        {
        }
        public virtual void Write(TNetObject innerNetObject, MessageWriter messageWriter)
        {
            Write(messageWriter);
        }
    }
}
using FungleAPI.Attributes;
using FungleAPI.Networking;
using Hazel;
using InnerNet;

namespace FungleAPI.Base.Rpc
{
    [FungleIgnore]
    public class AdvancedRpc<DataT> : BaseRpcHelper<InnerNetObject>
    {
        public void Send(DataT data, InnerNetObject innerNetObject, SendOption sendOption = SendOption.Reliable, int targetClientId = -1)
        {
            CustomRpcManager.SendRpc(innerNetObject, delegate (MessageWriter writer)
            {
                writer.WriteRPC(this);
                Write(innerNetObject, writer, data);
            }, sendOption, targetClientId);
        }
        public virtual void Write(MessageWriter messageWriter, DataT value)
        {
        }
        public virtual void Write(InnerNetObject innerNetObject, MessageWriter messageWriter, DataT value)
        {
            Write(messageWriter, value);
        }
    }
    [FungleIgnore]
    public class AdvancedRpc<DataT, TNetObject> : BaseRpcHelper<TNetObject> where TNetObject : InnerNetObject
    {
        public void Send(DataT data, TNetObject innerNetObject, SendOption sendOption = SendOption.Reliable, int targetClientId = -1)
        {
            CustomRpcManager.SendRpc(innerNetObject, delegate (MessageWriter writer)
            {
                writer.WriteRPC(this);
                Write(innerNetObject, writer, data);
            }, sendOption, targetClientId);
        }
        public virtual void Write(MessageWriter messageWriter, DataT value)
        {
        }
        public virtual void Write(TNetObject innerNetObject, MessageWriter messageWriter, DataT value)
        {
            Write(messageWriter, value);
        }
    }
}
// And this is a Example rpc
using FungleAPI.Base.Rpc;
using Hazel;
using UnityEngine;

public class ModRpc : AdvancedRpc<string, PlayerControl>
{
    public override void Write(PlayerControl innerNetObject, MessageWriter messageWriter, string value)
    {
        messageWriter.Write(value);
        Debug.Log($"{innerNetObject.name} sayed {value}.");
    }
    public override void Handle(PlayerControl innerNetObject, MessageReader messageReader)
    {
        string value = messageReader.ReadString();
        Debug.Log($"{innerNetObject.name} sayed {value}.");
    }
}
// You can get the Rpc Instance with
FungleAPI.Networking.CustomRpcManager.Instance<ModRpc>();
// And send it with
FungleAPI.Networking.CustomRpcManager.Instance<ModRpc>().Send("Hello!!", PlayerControl.LocalPlayer); // PlayerControl.LocalPlayer is a InnerNetObject
# Teams
// A team is used to separate players and help the GameOver
// You can create a team with
using System;
using System.Collections.Generic;
using System.Text;
using System.Reflection;
using AmongUs.GameOptions;
using FungleAPI;
using FungleAPI.Configuration;
using FungleAPI.Configuration.Attributes;
using FungleAPI.GameOver;
using FungleAPI.PluginLoading;
using FungleAPI.Teams;
using FungleAPI.Utilities;
using FungleAPI.Utilities.Prefabs;
using HarmonyLib;
using UnityEngine;

public class ModTeam : ModdedTeam
{
    public override Color TeamColor => Palette.CrewmateBlue; // Put here your team color
    public override StringNames TeamName => StringNames.None; // Put here your StringName from team name
    public override StringNames PluralName => StringNames.None; // Put here your StringName from plural team name
    public override bool FriendlyFire => true; // Team members can kill each other
    public override bool KnowMembers => false; // Team members know who is on their team
    public override CustomGameOver DefaultGameOver { get; } // The GameOver used
    public override string VictoryText { get; } // The victory text, optional if you have a DefaultGameOver
    public override uint MaxCount => 3; // Max number of players
    public override uint DefaultCount => 1; // Default number of players
    public override uint DefaultPriority => 1; // Default assyngnment priority
    public override RoleTypes DefaultRole { get; } // The default role assigned 
    public override bool AssignOnlyEnabledRoles => true; // Try to assign only enabled roles
    public override CategoryHeaderEditRole CreatCategoryHeaderEditRole(Transform parent) // Optional
    {
        CategoryHeaderEditRole categoryHeaderEditRole = UnityEngine.Object.Instantiate(PrefabUtils.Prefab<CategoryHeaderEditRole>(), Vector3.zero, Quaternion.identity, parent);
        categoryHeaderEditRole.SetHeader(StringNames.None, 20);
        categoryHeaderEditRole.Background.color = TeamColor.Light(0.7f);
        categoryHeaderEditRole.countLabel.color = TeamColor;
        categoryHeaderEditRole.chanceLabel.color = TeamColor;
        categoryHeaderEditRole.blankLabel.color = TeamColor.Dark(0.7f);
        categoryHeaderEditRole.Title.text = PluralName.GetString();
        categoryHeaderEditRole.Title.color = TeamColor.Light(0.9f);
        categoryHeaderEditRole.Title.enabled = true;
        return categoryHeaderEditRole;
    }
    public override CategoryHeaderRoleVariant CreateCategoryHeaderRoleVariant(Transform parent) // Optional
    {
        CategoryHeaderRoleVariant categoryHeaderRoleVariant = UnityEngine.Object.Instantiate(PrefabUtils.Prefab<CategoryHeaderRoleVariant>(), parent);
        categoryHeaderRoleVariant.SetHeader(StringNames.CrewmateRolesHeader, 61);
        string[] names = StringNames.CrewmateRolesHeader.GetString().Split(" ");
        categoryHeaderRoleVariant.Title.text = names[0] + " " + names[1] + " " + TeamName.GetString();
        categoryHeaderRoleVariant.Background.color = TeamColor.Light();
        categoryHeaderRoleVariant.Title.color = categoryHeaderRoleVariant.Background.color.Dark();
        categoryHeaderRoleVariant.Title.enabled = true;
        for (int i = 2; i < categoryHeaderRoleVariant.transform.GetChildCount(); i++)
        {
            categoryHeaderRoleVariant.transform.GetChild(i).gameObject.SetActive(false);
        }
        return categoryHeaderRoleVariant;
    }
    public override void Initialize(ModPlugin plugin) // Optional
    {
        if (!initialized)
        {
            Type type = GetType();
            ExtraConfigs = new List<ModdedOption>();
            foreach (PropertyInfo property in type.GetProperties())
            {
                ModdedOption att = (ModdedOption)property.GetCustomAttribute(typeof(ModdedOption));
                if (att != null)
                {
                    att.Initialize(property);
                    MethodInfo method = property.GetGetMethod(true);
                    if (method != null)
                    {
                        ConfigurationManager.Options.Add(att);
                        plugin.Options.Add(att);
                        HarmonyHelper.Patches.Add(method, new Func<object>(att.GetReturnedValue));
                        FungleAPIPlugin.Harmony.Patch(method, new HarmonyMethod(typeof(HarmonyHelper).GetMethod("GetPrefix", BindingFlags.Static | BindingFlags.Public)));
                    }
                    ExtraConfigs.Add(att);
                }
            }
            initialized = true;
        }
    }
    public override OptionBehaviour CreateCountOption(Transform transform) // Optional
    {
        NumberOption numberOption = UnityEngine.Object.Instantiate(PrefabUtils.Prefab<NumberOption>(), Vector3.zero, Quaternion.identity, transform);
        numberOption.SetUpFromData(CountData, 20);
        numberOption.OnValueChanged = new Action<OptionBehaviour>(delegate
        {
            if (this == Impostors)
            {
                GameOptionsManager.Instance.currentGameOptions.SetInt(Int32OptionNames.NumImpostors, (int)numberOption.Value);
            }
            CountAndPriority.SetCount((int)numberOption.Value);
        });
        numberOption.Title = CountData.Title;
        numberOption.Value = CountAndPriority.GetCount();
        numberOption.oldValue = numberOption.oldValue;
        numberOption.Increment = CountData.Increment;
        numberOption.ValidRange = CountData.ValidRange;
        numberOption.FormatString = CountData.FormatString;
        numberOption.ZeroIsInfinity = CountData.ZeroIsInfinity;
        numberOption.SuffixType = CountData.SuffixType;
        numberOption.floatOptionName = FloatOptionNames.Invalid;
        ModdedOption.FixOption(numberOption);
        return numberOption;
    }
    public override OptionBehaviour CreatePriorityOption(Transform transform) // Optional
    {
        NumberOption numberOption = UnityEngine.Object.Instantiate(PrefabUtils.Prefab<NumberOption>(), Vector3.zero, Quaternion.identity, transform);
        numberOption.SetUpFromData(CountData, 20);
        numberOption.OnValueChanged = new Action<OptionBehaviour>(delegate
        {
            CountAndPriority.SetPriority((int)numberOption.Value);
        });
        numberOption.Title = PriorityData.Title;
        numberOption.Value = CountAndPriority.GetPriority();
        numberOption.oldValue = numberOption.oldValue;
        numberOption.Increment = PriorityData.Increment;
        numberOption.ValidRange = PriorityData.ValidRange;
        numberOption.FormatString = PriorityData.FormatString;
        numberOption.ZeroIsInfinity = PriorityData.ZeroIsInfinity;
        numberOption.SuffixType = PriorityData.SuffixType;
        numberOption.floatOptionName = FloatOptionNames.Invalid;
        ModdedOption.FixOption(numberOption);
        return numberOption;
    }
}
// You can get the Team Instance with
FungleAPI.Teams.ModdedTeam.Instance<ModTeam>();
// And if its a Vanilla team use
FungleAPI.Teams.ModdedTeam.Crewmates;
FungleAPI.Teams.ModdedTeam.Impostors;
FungleAPI.Teams.ModdedTeam.Neutrals;
# Roles
// Roles are the highlight of the mod and the game, they are what determine the direction of the game
// First you need to know how the ICustomRole Interface works
using AmongUs.GameOptions;
using System;
using System.Collections.Generic;
using System.Text;
using UnityEngine;
using FungleAPI.Utilities;
using FungleAPI.Configuration.Attributes;
using FungleAPI.Configuration.Helpers;
using FungleAPI.Teams;

namespace FungleAPI.Role
{
    public interface ICustomRole
    {
        ModdedTeam Team { get; } // The role team
        StringNames RoleName { get; } // The StringName from role name
        StringNames RoleBlur { get; } // The StringName from role blur
        StringNames RoleBlurMed { get; } // The StringName from role blur med
        StringNames RoleBlurLong { get; } // The StringName from role blur long
        Color RoleColor { get; } // The role color
        List<ModdedOption> Options { get { return Save[GetType()].Options.Value; } } // Do not use if you don't know what you're doing
        RoleCountAndChance CountAndChance { get { return Save[GetType()].CountAndChance.Value; } } // Do not use if you don't know what you're doing
        string ExileText(ExileController exileController) // The text showed when the role is ejected
        {
            string[] tx = StringNames.ExileTextSP.GetString().Split(' ', StringSplitOptions.RemoveEmptyEntries);
            return exileController.initData.networkedPlayer.PlayerName + " " + tx[1] + " " + tx[2] + " " + exileController.initData.networkedPlayer.Role.NiceName;
        }
        SabotageButtonConfig CreateSabotageConfig() => null;
        ReportButtonConfig CreateReportConfig() => null;
        VentButtonConfig CreateVentConfig() => null;
        KillButtonConfig CreateKillConfig() => null;
        MiraRoleTabConfig RoleTabConfig => new MiraRoleTabConfig(this); // Do not use if you don't know what you're doing
        RoleHintType HintType => RoleHintType.TaskHint; // How the role name and description are showed
        bool CanUseVent => Team == ModdedTeam.Impostors; // If role can use vents
        bool IsAffectedByLightOnAirship => Team == ModdedTeam.Impostors;
        bool UseVanillaKillButton => Team == ModdedTeam.Impostors; // If role can use the vanilla kill button
        bool CanSabotage => Team == ModdedTeam.Impostors; // If role can sabotage
        bool CompletedTasksCountForProgress => Team == ModdedTeam.Crewmates; // If role tasks will count for the Crewmate Win By Tasks
        bool IsGhostRole => false; // If role is a ghost
        RoleTypes GhostRole => Team == ModdedTeam.Crewmates ? RoleTypes.CrewmateGhost : (Team == ModdedTeam.Impostors ? RoleTypes.ImpostorGhost : CustomRoleManager.NeutralGhost.Role); // When killed what will be the ghost role
        List<RoleTypes> AvaibleGhostRoles => null; // Possible ghosts roles
        bool ShowTeamColor => false; // Show on TeamColor property the role color
        Sprite Screenshot => null; // The screenshot that will apper on role configuration (Lobby)
        bool HideRole => false; // If role will not appear and cannot be assigned on game start
        bool HideInFreeplayComputer => false; // If role will not apper in freeplay computer
        int MaxRoleCount => 15;
        Sprite IconSolid => null; 
        Sprite IconWhite => null;
        DeadBodyType CreatedDeadBodyOnKill => DeadBodyType.Normal; // The dead body that will be created when the role kill
        string NeutralWinText => "Victory of the " + RoleName.GetString();
        public RoleTypes Role => CustomRoleManager.RolesToRegister[GetType()]; // Do not use
        bool CanKill => UseVanillaKillButton; // If role can kill
        public Color OutlineColor => RoleColor; // The Outline color
        internal static Dictionary<Type, (ChangeableValue<List<ModdedOption>> Options, ChangeableValue<RoleCountAndChance> CountAndChance)> Save = new(); // Do not use
    }
}
// Configs like SabotageButtonConfig, ReportButtonConfig, VentButtonConfig, KillButtonConfig are the way to Configurate the Vanilla hud buttons
using System;
using UnityEngine;

namespace FungleAPI.Role
{
    public class SabotageButtonConfig
    {
        public static Sprite DefaultSprite;
        public SabotageButtonConfig()
        {
            Cooldown = () => 0;
            CanUse = () => Button.isActiveAndEnabled && !Button.IsOnCooldown;
            Refresh = delegate
            {
                PlayerControl player = PlayerControl.LocalPlayer;
                if (GameManager.Instance == null || player == null)
                {
                    Button.ToggleVisible(false);
                    Button.SetDisabled();
                    return;
                }
                if (player.inVent || !GameManager.Instance.SabotagesEnabled() || player.petting)
                {
                    Button.ToggleVisible(player.Data.Role.CanSabotage() && GameManager.Instance.SabotagesEnabled());
                }
                if (Button.isActiveAndEnabled)
                {
                    if (CanUse())
                    {
                        Button.SetEnabled();
                        return;
                    }
                    Button.SetDisabled();
                }
            };
            SetTimer = delegate (float time)
            {
                float cooldown = Cooldown();
                if (cooldown <= 0)
                {
                    return;
                }
                Timer = Mathf.Clamp(time, 0, cooldown);
                Button.SetCoolDown(Timer, cooldown);
            };
            ResetTimer = delegate (bool doors)
            {
                if (!doors)
                {
                    SetTimer(Cooldown());
                    if (Timer > 0)
                    {
                        MapBehaviour.Instance?.Close();
                    }
                }
            };
            DoClick = delegate
            {
                if (PlayerControl.LocalPlayer.Data.Role.CanSabotage() && CanUse() && !PlayerControl.LocalPlayer.inVent && GameManager.Instance.SabotagesEnabled())
                {
                    HudManager.Instance.ToggleMapVisible(new MapOptions
                    {
                        Mode = MapOptions.Modes.Sabotage
                    });
                }
            };
            ResetButton = delegate
            {
                SetTimer(0);
                Button.buttonLabelText.GetComponent<TextTranslatorTMP>().TargetText = StringNames.SabotageLabel;
                Button.buttonLabelText.GetComponent<TextTranslatorTMP>().ResetText();
                Button.buttonLabelText.SetOutlineColor(Palette.ImpostorRed);
                Button.graphic.SetCooldownNormalizedUvs();
                Button.graphic.sprite = DefaultSprite;
            };
            Update = delegate
            {
                SetTimer(Timer - Time.deltaTime);
            };
        }
        public Func<float> Cooldown;
        public Func<bool> CanUse;
        public Action<float> SetTimer;
        public Action<bool> ResetTimer;
        public Action Refresh;
        public Action DoClick;
        public Action Update;
        public Action ResetButton;
        public Action InitializeButton;
        public float Timer;
        public SabotageButton Button => HudManager.Instance.SabotageButton;
    }
}
using System;
using UnityEngine;

namespace FungleAPI.Role
{
    public class ReportButtonConfig
    {
        public static Sprite DefaultSprite;
        public ReportButtonConfig()
        {
            CanUse = () => true;
            SetActive = delegate (bool isActive)
            {
                if (GameManager.Instance == null)
                {
                    Button.SetDisabled();
                    Button.ToggleVisible(false);
                    return;
                }
                if (!GameManager.Instance.CanReportBodies())
                {
                    Button.ToggleVisible(false);
                    return;
                }
                if (isActive && CanUse())
                {
                    Button.SetEnabled();
                    return;
                }
                Button.SetDisabled();
            };
            DoClick = delegate
            {
                if (Button.isActiveAndEnabled && CanUse() && GameManager.Instance.CanReportBodies())
                {
                    PlayerControl.LocalPlayer.ReportClosest();
                }
            };
            ResetButton = delegate
            {
                Button.buttonLabelText.GetComponent<TextTranslatorTMP>().TargetText = StringNames.ReportLabel;
                Button.buttonLabelText.GetComponent<TextTranslatorTMP>().ResetText();
                Button.buttonLabelText.SetOutlineColor(Color.black);
                Button.graphic.sprite = DefaultSprite;
            };
        }
        public Func<bool> CanUse;
        public Action<bool> SetActive;
        public Action DoClick;
        public Action ResetButton;
        public Action InitializeButton;
        public ReportButton Button => HudManager.Instance.ReportButton;
    }
}
using System;
using UnityEngine;

namespace FungleAPI.Role
{
    public class VentButtonConfig
    {
        public static Sprite DefaultSprite;
        public VentButtonConfig()
        {
            CanUse = () => Button.isActiveAndEnabled && Button.currentTarget != null && !Button.IsOnCooldown && !PlayerControl.LocalPlayer.Data.IsDead;
            Cooldown = () => 0;
            ShowOutline = () => Button.isActiveAndEnabled && !Button.IsOnCooldown && !PlayerControl.LocalPlayer.Data.IsDead;
            SetTarget = delegate (Vent target)
            {
                Button.currentTarget = target;
            };
            SetTimer = delegate (float time)
            {
                float cooldown = Cooldown();
                if (cooldown <= 0)
                {
                    return;
                }
                Timer = Mathf.Clamp(time, 0, cooldown);
                Button.SetCoolDown(Timer, cooldown);
            };
            DoClick = delegate
            {
                if (CanUse())
                {
                    if (PlayerControl.LocalPlayer.inVent && !PlayerControl.LocalPlayer.walkingToVent)
                    {
                        Timer = Cooldown();
                    }
                    Button.currentTarget.Use();
                }
            };
            ResetButton = delegate
            {
                SetTimer(0);
                Button.buttonLabelText.GetComponent<TextTranslatorTMP>().TargetText = StringNames.VentLabel;
                Button.buttonLabelText.GetComponent<TextTranslatorTMP>().ResetText();
                Button.buttonLabelText.SetOutlineColor(Palette.ImpostorRed);
                Button.graphic.SetCooldownNormalizedUvs();
                Button.graphic.sprite = DefaultSprite;
            };
            bool enabled = true;
            Update = delegate
            {
                SetTimer(Timer - Time.deltaTime);
                if (enabled != CanUse())
                {
                    enabled = CanUse();
                    if (enabled)
                    {
                        Button.SetEnabled();
                        return;
                    }
                    Button.SetDisabled();
                }
            };
        }
        public Func<bool> CanUse;
        public Func<float> Cooldown;
        public Func<bool> ShowOutline;
        public Action<Vent> SetTarget;
        public Action<float> SetTimer;
        public Action DoClick;
        public Action Update;
        public Action ResetButton;
        public Action InitializeButton;
        public float Timer;
        public VentButton Button => HudManager.Instance.ImpostorVentButton;
    }
}
using System;
using UnityEngine;

namespace FungleAPI.Role
{
    public class KillButtonConfig
    {
        public KillButtonConfig()
        {
            CanUse = () => Button.isActiveAndEnabled && Button.currentTarget != null && !Button.isCoolingDown && !PlayerControl.LocalPlayer.Data.IsDead && PlayerControl.LocalPlayer.CanMove;
            Cooldown = () => GameOptionsManager.Instance.CurrentGameOptions.GetFloat(AmongUs.GameOptions.FloatOptionNames.KillCooldown);
            CheckClick = delegate (PlayerControl target)
            {
                if (Button.currentTarget && Button.currentTarget == target)
                {
                    DoClick();
                    return;
                }
                if (Button.currentTarget && Button.currentTarget != target && PlayerControl.LocalPlayer.Data.Role.GetPlayersInAbilityRangeSorted(new Il2CppSystem.Collections.Generic.List<PlayerControl>()).Contains(target))
                {
                    SetTarget(target);
                    DoClick();
                }
            };
            SetTarget = delegate (PlayerControl target)
            {
                if (!PlayerControl.LocalPlayer || PlayerControl.LocalPlayer.Data == null || !PlayerControl.LocalPlayer.Data.Role)
                {
                    return;
                }
                ICustomRole customRole = PlayerControl.LocalPlayer.Data.Role.CustomRole();
                if (Button.currentTarget && Button.currentTarget != target)
                {
                    Button.currentTarget.cosmetics.SetOutline(false, new Il2CppSystem.Nullable<Color>(Color.clear));
                }
                Button.currentTarget = target;
                if (Button.currentTarget)
                {
                    if (customRole != null)
                    {
                        Button.currentTarget.cosmetics.SetOutline(true, new Il2CppSystem.Nullable<Color>(customRole.OutlineColor));
                        return;
                    }
                    target.ToggleHighlight(true, PlayerControl.LocalPlayer.Data.Role.TeamType);
                }
            };
            DoClick = delegate
            {
                if (CanUse())
                {
                    PlayerControl.LocalPlayer.CmdCheckMurder(Button.currentTarget);
                    SetTarget(null);
                }
            };
            ResetButton = delegate
            {
                PlayerControl.LocalPlayer?.SetKillTimer(10f);
                Button.ChangeButtonText(StringNames.KillLabel);
                Button.graphic.sprite = Button.defaultKillSprite;
                Button.graphic.SetCooldownNormalizedUvs();
                Button.buttonLabelText.SetOutlineColor(Palette.ImpostorRed);
            };
            bool enabled = true;
            Update = delegate
            {
                if (CanUse() != enabled)
                {
                    enabled = CanUse();
                    if (enabled)
                    {
                        Button.SetEnabled();
                        return;
                    }
                    Button.SetDisabled();

                }
            };
        }
        public Func<bool> CanUse;
        public Func<float> Cooldown;
        public Action<PlayerControl> CheckClick;
        public Action<PlayerControl> SetTarget;
        public Action DoClick;
        public Action Update;
        public Action ResetButton;
        public Action InitializeButton;
        public KillButton Button => HudManager.Instance.KillButton;
    }
}
// CustomRoleManager
using AmongUs.GameOptions;
using FungleAPI.Configuration;
using FungleAPI.Configuration.Attributes;
using FungleAPI.Configuration.Helpers;
using FungleAPI.PluginLoading;
using FungleAPI.Teams;
using FungleAPI.Translation;
using FungleAPI.Utilities;
using HarmonyLib;
using Il2CppInterop.Runtime;
using System;
using System.Collections.Generic;
using System.Reflection;
using System.Text;
using UnityEngine;

namespace FungleAPI.Role
{
    public static class CustomRoleManager
    {
        public static MiraRoleTabConfig CurrentRoleTabConfig;
        public static KillButtonConfig CurrentKillConfig;
        public static VentButtonConfig CurrentVentConfig;
        public static ReportButtonConfig CurrentReportConfig;
        public static SabotageButtonConfig CurrentSabotageConfig;
        public static RoleBehaviour NeutralGhost => Instance<NeutralGhost>();
        public static List<RoleBehaviour> AllRoles = new List<RoleBehaviour>();
        public static List<ICustomRole> AllCustomRoles = new List<ICustomRole>();
        internal static int id = Enum.GetNames<RoleTypes>().Length + 20;
        internal static Dictionary<Type, RoleTypes> RolesToRegister = new Dictionary<Type, RoleTypes>();
        public static T Instance<T>() where T : RoleBehaviour
        {
            foreach (RoleBehaviour role in RoleManager.Instance.AllRoles)
            {
                if (role.GetType() == typeof(T))
                {
                    return role.SafeCast<T>();
                }
            }
            return null;
        }
        public static RoleTypes GetType<T>() where T : RoleBehaviour
        {
            return RolesToRegister[typeof(T)];
        }
        public static RoleTypes GetType(Type type)
        {
            return RolesToRegister[type];
        }
        public static ICustomRole CustomRole(this RoleBehaviour role)
        {
            return role as ICustomRole;
        }
        public static ICustomRole GetRole(RoleTypes type)
        {
            return RoleManager.Instance.GetRole(type).CustomRole();
        }
        public static int CaculeCountByChance(this RoleBehaviour role, IRoleOptionsCollection roleOptions)
        {
            int count = 0;
            for (int i = 0; i < roleOptions.GetNumPerGame(role.Role); i++)
            {
                if (new System.Random().Next(0, 100) < roleOptions.GetChancePerGame(role.Role))
                {
                    count++;
                }
            }
            return count;
        }
        public static void UpdateRole(RoleBehaviour role)
        {
            if (role == null)
            {
                return;
            }
            bool amOwner = role.Player.AmOwner;
            ICustomRole customRole = role.CustomRole();
            if (customRole != null)
            {
                role.StringName = customRole.RoleName;
                role.BlurbName = customRole.RoleBlur;
                role.BlurbNameMed = customRole.RoleBlurMed;
                role.BlurbNameLong = customRole.RoleBlurLong;
                role.NameColor = customRole.RoleColor;
                role.AffectedByLightAffectors = customRole.IsAffectedByLightOnAirship;
                role.CanUseKillButton = customRole.UseVanillaKillButton;
                role.CanVent = customRole.CanUseVent;
                role.TasksCountTowardProgress = customRole.CompletedTasksCountForProgress;
                role.RoleScreenshot = customRole.Screenshot;
                role.RoleIconSolid = customRole.IconSolid;
                role.RoleIconWhite = customRole.IconWhite;
                if (amOwner)
                {
                    CurrentRoleTabConfig = customRole.RoleTabConfig;
                    CurrentKillConfig = customRole.CreateKillConfig();
                    CurrentVentConfig = customRole.CreateVentConfig();
                    CurrentReportConfig = customRole.CreateReportConfig();
                    CurrentSabotageConfig = customRole.CreateSabotageConfig();
                }
            }
            else if (amOwner)
            {
                CurrentRoleTabConfig = null;
                CurrentKillConfig = null;
                CurrentVentConfig = null;
                CurrentReportConfig = null;
                CurrentSabotageConfig = null;
            }
            if (amOwner)
            {
                if (CurrentRoleTabConfig == null)
                {
                    CurrentRoleTabConfig = new MiraRoleTabConfig() { __text = role.NiceName, TabNameColor = role.TeamColor };
                }
                if (CurrentKillConfig == null)
                {
                    CurrentKillConfig = new KillButtonConfig();
                }
                if (CurrentVentConfig == null)
                {
                    CurrentVentConfig = new VentButtonConfig();
                }
                if (CurrentReportConfig == null)
                {
                    CurrentReportConfig = new ReportButtonConfig();
                }
                if (CurrentSabotageConfig == null)
                {
                    CurrentSabotageConfig = new SabotageButtonConfig();
                }
            }
        }
        internal static RoleBehaviour Register(Type type, ModPlugin plugin, RoleTypes roleType)
        {
            (ChangeableValue<List<ModdedOption>> Options, ChangeableValue<RoleCountAndChance> CountAndChance) pair = ICustomRole.Save[type];
            RoleBehaviour role = (RoleBehaviour)new GameObject().AddComponent(Il2CppType.From(type)).DontDestroy();
            ICustomRole customRole = role.CustomRole();
            InitializeRoleOptions(role, plugin);
            ConfigurationManager.InitializeRoleCountAndChances(type, plugin);
            role.name = type.Name;
            role.StringName = customRole.RoleName;
            role.BlurbName = customRole.RoleBlur;
            role.BlurbNameMed = customRole.RoleBlurMed;
            role.BlurbNameLong = customRole.RoleBlurLong;
            role.NameColor = customRole.RoleColor;
            role.AffectedByLightAffectors = customRole.IsAffectedByLightOnAirship;
            role.CanUseKillButton = customRole.UseVanillaKillButton;
            role.CanVent = customRole.CanUseVent;
            role.TasksCountTowardProgress = customRole.CompletedTasksCountForProgress;
            role.RoleScreenshot = customRole.Screenshot;
            role.RoleIconSolid = customRole.IconSolid;
            role.RoleIconWhite = customRole.IconWhite;
            role.Role = roleType;
            role.TeamType = customRole.Team == ModdedTeam.Impostors ? RoleTeamTypes.Impostor : RoleTeamTypes.Crewmate;
            AllRoles.Add(role);
            AllCustomRoles.Add(customRole);
            plugin.Roles.Add(role);
            if (customRole.IsGhostRole)
            {
                RoleManager.GhostRoles.Add(roleType);
            }
            return role;
        }
        internal static void InitializeRoleOptions(RoleBehaviour obj, ModPlugin plugin)
        {
            Type type = obj.GetType();
            foreach (PropertyInfo property in type.GetProperties())
            {
                ModdedOption att = (ModdedOption)property.GetCustomAttribute(typeof(ModdedOption));
                if (att != null)
                {
                    att.Initialize(property);
                    MethodInfo method = property.GetGetMethod(true);
                    if (method != null)
                    {
                        ConfigurationManager.Options.Add(att);
                        plugin.Options.Add(att);
                        HarmonyHelper.Patches.Add(method, new Func<object>(att.GetReturnedValue));
                        FungleAPIPlugin.Harmony.Patch(method, new HarmonyMethod(typeof(HarmonyHelper).GetMethod("GetPrefix", BindingFlags.Static | BindingFlags.Public)));
                    }
                    if (obj.CustomRole() != null)
                    {
                        obj.CustomRole().Options.Add(att);
                    }
                }
            }
        }
        public static void CreateForRole(Il2CppSystem.Text.StringBuilder sb, RoleBehaviour role, Color? color = null)
        {
            if (color == null)
            {
                color = role.TeamColor;
            }
            sb.AppendLine(color.Value.ToTextColor() + FungleTranslation.YourRoleIsText.GetString() + "<b>" + role.NiceName + "</b>.</color>");
            sb.AppendLine("<size=70%>" + role.BlurbLong);
        }
        public static ModPlugin GetRolePlugin(this RoleBehaviour role)
        {
            Assembly assembly = role.GetType().Assembly;
            foreach (ModPlugin plugin in ModPlugin.AllPlugins)
            {
                if (plugin.ModAssembly == assembly)
                {
                    return plugin;
                }
            }
            return FungleAPIPlugin.Plugin;
        }
        public static RoleHintType GetHintType(this RoleBehaviour role)
        {
            if (role.CustomRole() != null)
            {
                return role.CustomRole().HintType;
            }
            return RoleHintType.TaskHint;
        }
        public static ModdedTeam GetTeam(this RoleBehaviour role)
        {
            if (role.CustomRole() != null)
            {
                return role.CustomRole().Team;
            }
            return RoleManager.IsImpostorRole(role.Role) ? ModdedTeam.Impostors : ModdedTeam.Crewmates;
        }
        public static bool CanSabotage(this RoleBehaviour roleBehaviour)
        {
            if (roleBehaviour.CustomRole() != null)
            {
                return roleBehaviour.CustomRole().CanSabotage;
            }
            return roleBehaviour.TeamType == RoleTeamTypes.Impostor;
        }
        public static bool CanKill(this RoleBehaviour roleBehaviour)
        {
            if (roleBehaviour.CustomRole() != null)
            {
                return roleBehaviour.CustomRole().CanKill;
            }
            return roleBehaviour.CanUseKillButton;
        }
        public static bool UseKillButton(this RoleBehaviour roleBehaviour)
        {
            if (roleBehaviour.CustomRole() != null)
            {
                return roleBehaviour.CustomRole().UseVanillaKillButton;
            }
            return roleBehaviour.CanUseKillButton;
        }
        public static bool CanVent(this RoleBehaviour roleBehaviour)
        {
            if (roleBehaviour.CustomRole() != null)
            {
                return roleBehaviour.CustomRole().CanUseVent;
            }
            return roleBehaviour.CanVent;
        }
    }
}
// You have some role base to use and help you
using FungleAPI.Attributes;
using FungleAPI.Player;
using FungleAPI.Role;
using FungleAPI.Utilities;

namespace FungleAPI.Base.Roles
{
    [FungleIgnore]
    public class CrewmateBase : RoleBehaviour
    {
        public virtual bool DoTasks => false;
        public override bool IsDead => false;
        public override bool CanUse(IUsable usable)
        {
            if (usable.SafeCast<ZiplineConsole>() != null || usable.SafeCast<Ladder>() != null || usable.SafeCast<PlatformConsole>() != null || usable.SafeCast<Console>() != null || usable.SafeCast<DoorConsole>() != null)
            {
                return DoTasks;
            }
            return true;
        }
        public override bool IsValidTarget(NetworkedPlayerInfo target)
        {
            return !(target == null) && !target.Disconnected && !target.IsDead && target.PlayerId != this.Player.PlayerId && !(target.Role == null) && !(target.Object == null) && !target.Object.inVent && !target.Object.inMovingPlat && target.Object.Visible;
        }
        public override PlayerControl FindClosestTarget()
        {
            Il2CppSystem.Collections.Generic.List<PlayerControl> playersInAbilityRangeSorted = GetPlayersInAbilityRangeSorted(GetTempPlayerList());
            if (playersInAbilityRangeSorted.Count <= 0)
            {
                return null;
            }
            return playersInAbilityRangeSorted[0];
        }
        public override bool DidWin(GameOverReason gameOverReason)
        {
            return GameManager.Instance.DidHumansWin(gameOverReason);
        }
        public override DeadBody FindClosestBody()
        {
            return Player.GetClosestDeadBody(GetAbilityDistance());
        }
        public override void AppendTaskHint(Il2CppSystem.Text.StringBuilder taskStringBuilder)
        {
            if (this.GetHintType() == RoleHintType.MiraAPI_RoleTab)
            {
                CustomRoleManager.CreateForRole(taskStringBuilder, this);
                return;
            }
            base.AppendTaskHint(taskStringBuilder);
        }
    }
}
using AmongUs.GameOptions;
using FungleAPI.Attributes;
using FungleAPI.Player;
using FungleAPI.Role;
using FungleAPI.Utilities;
using System.Linq;
using UnityEngine;

namespace FungleAPI.Base.Roles
{
    [FungleIgnore]
    public class ImpostorBase : RoleBehaviour
    {
        public virtual bool DoTasks => false;
        public override bool IsDead => false;
        public override void Deinitialize(PlayerControl targetPlayer)
        {
            PlayerTask playerTask = targetPlayer.myTasks.ToSystemList().FirstOrDefault((PlayerTask t) => t.name == "ImpostorRole");
            if (playerTask)
            {
                targetPlayer.myTasks.Remove(playerTask);
                GameObject.Destroy(playerTask.gameObject);
            }
        }
        public override void SpawnTaskHeader(PlayerControl playerControl)
        {
            if (playerControl != PlayerControl.LocalPlayer)
            {
                return;
            }
            ImportantTextTask orCreateTask = PlayerTask.GetOrCreateTask<ImportantTextTask>(playerControl, 0);
            switch (GameOptionsManager.Instance.CurrentGameOptions.GameMode)
            {
                case GameModes.Normal:
                case GameModes.NormalFools:
                    orCreateTask.Text = string.Concat(new string[]
                    {
                DestroyableSingleton<TranslationController>.Instance.GetString(StringNames.ImpostorTask),
                "\r\n",
                Palette.ImpostorRed.ToTextColor(),
                DestroyableSingleton<TranslationController>.Instance.GetString(StringNames.FakeTasks),
                "</color>"
                    });
                    return;
                case GameModes.HideNSeek:
                case GameModes.SeekFools:
                    orCreateTask.Text = DestroyableSingleton<TranslationController>.Instance.GetString(StringNames.RuleOneImpostor);
                    return;
                default:
                    return;
            }
        }
        public override bool CanUse(IUsable usable)
        {
            if (!GameManager.Instance.LogicUsables.CanUse(usable, Player))
            {
                return false;
            }
            Console console = usable.SafeCast<Console>();
            return console == null || console.AllowImpostor || DoTasks;
        }
        public override bool IsValidTarget(NetworkedPlayerInfo target)
        {
            return !(target == null) && !target.Disconnected && !target.IsDead && target.PlayerId != this.Player.PlayerId && !(target.Role == null) && !(target.Object == null) && !target.Object.inVent && !target.Object.inMovingPlat && target.Object.Visible && (target.Role.GetTeam() != this.GetTeam() || this.GetTeam().FriendlyFire);
        }
        public override PlayerControl FindClosestTarget()
        {
            Il2CppSystem.Collections.Generic.List<PlayerControl> playersInAbilityRangeSorted = GetPlayersInAbilityRangeSorted(GetTempPlayerList());
            if (playersInAbilityRangeSorted.Count <= 0)
            {
                return null;
            }
            return playersInAbilityRangeSorted[0];
        }
        public override bool DidWin(GameOverReason gameOverReason)
        {
            return GameManager.Instance.DidImpostorsWin(gameOverReason);
        }
        public override DeadBody FindClosestBody()
        {
            return Player.GetClosestDeadBody(GetAbilityDistance());
        }
        public override void AppendTaskHint(Il2CppSystem.Text.StringBuilder taskStringBuilder)
        {
            if (this.GetHintType() == RoleHintType.MiraAPI_RoleTab)
            {
                CustomRoleManager.CreateForRole(taskStringBuilder, this);
                return;
            }
            base.AppendTaskHint(taskStringBuilder);
        }
    }
}
using AmongUs.GameOptions;
using FungleAPI.Attributes;
using FungleAPI.GameOver;
using FungleAPI.GameOver.Ends;
using FungleAPI.Player;
using FungleAPI.Role;
using FungleAPI.Utilities;
using System.Linq;
using UnityEngine;

namespace FungleAPI.Base.Roles
{
    [FungleIgnore]
    public class NeutralBase : RoleBehaviour
    {
        public virtual bool DoTasks => false;
        public override bool IsDead => false;
        public override void Deinitialize(PlayerControl targetPlayer)
        {
            PlayerTask playerTask = targetPlayer.myTasks.ToSystemList().FirstOrDefault((PlayerTask t) => t.name == "ImpostorRole");
            if (playerTask)
            {
                targetPlayer.myTasks.Remove(playerTask);
                GameObject.Destroy(playerTask.gameObject);
            }
        }
        public override void SpawnTaskHeader(PlayerControl playerControl)
        {
            if (playerControl != PlayerControl.LocalPlayer || !DoTasks)
            {
                return;
            }
            ImportantTextTask orCreateTask = PlayerTask.GetOrCreateTask<ImportantTextTask>(playerControl, 0);
            switch (GameOptionsManager.Instance.CurrentGameOptions.GameMode)
            {
                case GameModes.Normal:
                case GameModes.NormalFools:
                    orCreateTask.Text = string.Concat(new string[]
                    {
                DestroyableSingleton<TranslationController>.Instance.GetString(StringNames.ImpostorTask),
                "\r\n",
                Palette.ImpostorRed.ToTextColor(),
                DestroyableSingleton<TranslationController>.Instance.GetString(StringNames.FakeTasks),
                "</color>"
                    });
                    return;
                case GameModes.HideNSeek:
                case GameModes.SeekFools:
                    orCreateTask.Text = DestroyableSingleton<TranslationController>.Instance.GetString(StringNames.RuleOneImpostor);
                    return;
                default:
                    return;
            }
        }
        public override bool CanUse(IUsable usable)
        {
            if (!GameManager.Instance.LogicUsables.CanUse(usable, this.Player))
            {
                return false;
            }
            Console console = usable.SafeCast<Console>();
            return console == null || DoTasks;
        }
        public override bool IsValidTarget(NetworkedPlayerInfo target)
        {
            return !(target == null) && !target.Disconnected && !target.IsDead && target.PlayerId != this.Player.PlayerId && !(target.Role == null) && !(target.Object == null) && !target.Object.inVent && !target.Object.inMovingPlat && target.Object.Visible;
        }
        public override PlayerControl FindClosestTarget()
        {
            Il2CppSystem.Collections.Generic.List<PlayerControl> playersInAbilityRangeSorted = GetPlayersInAbilityRangeSorted(GetTempPlayerList());
            if (playersInAbilityRangeSorted.Count <= 0)
            {
                return null;
            }
            return playersInAbilityRangeSorted[0];
        }
        public override bool DidWin(GameOverReason gameOverReason)
        {
            return GameOverManager.Instance<NeutralGameOver>().Reason == gameOverReason;
        }
        public override DeadBody FindClosestBody()
        {
            return Player.GetClosestDeadBody(GetAbilityDistance());
        }
        public override void AppendTaskHint(Il2CppSystem.Text.StringBuilder taskStringBuilder)
        {
            if (this.GetHintType() == RoleHintType.MiraAPI_RoleTab)
            {
                CustomRoleManager.CreateForRole(taskStringBuilder, this);
                return;
            }
            base.AppendTaskHint(taskStringBuilder);
        }
    }
}
// The bases don't have the ICustomRole interface, but you need to include it
// How to create a role
using FungleAPI.Base.Roles;
using FungleAPI.Role;
using FungleAPI.Teams;
using FungleAPI.Translation;
using UnityEngine;

public class ModRole : CrewmateBase, ICustomRole
{
    public ModdedTeam Team { get; } = ModdedTeam.Crewmates;
    public StringNames RoleName { get; } = new Translator("your_role_name").StringName; // This will be explained later
    public StringNames RoleBlur { get; } = StringNames.None;
    public StringNames RoleBlurMed { get; } = StringNames.None;
    public StringNames RoleBlurLong { get; } = StringNames.None;
    public Color RoleColor { get; } = Color.cyan;
}
# Buttons
// Custom hud buttons
// CustomAbilityButton class
using FungleAPI.Attributes;
using FungleAPI.Utilities;
using System;
using System.Collections.Generic;
using UnityEngine;
namespace FungleAPI.Hud
{
    [FungleIgnore]
    public class CustomAbilityButton
    {
        internal static Dictionary<Type, CustomAbilityButton> Buttons = new Dictionary<Type, CustomAbilityButton>();
        public static T Instance<T>() where T : CustomAbilityButton
        {
            CustomAbilityButton button;
            Buttons.TryGetValue(typeof(T), out button);
            return button.SimpleCast<T>();
        }
        public AbilityButton Button;
        public virtual ButtonLocation Location => ButtonLocation.BottomRight; // Where button will stay
        public virtual bool Active => false; // Button will active
        public virtual bool CanClick { get; } // If you can click on
        public virtual bool CanUse { get; } // If you can use
        public virtual float Cooldown { get; } // The button default cooldown
        public virtual float InitialCooldown => Cooldown / 2; // The button initial cooldown
        public float Timer;
        public virtual bool HaveUses { get; } // Button has uses
        public virtual int NumUses { get; } // Number of uses
        public int CurrentNumUses;
        public virtual bool TransformButton { get; }
        public virtual float TransformDuration { get; }
        public float TransformTimer;
        public bool Transformed;
        public virtual void Destransform() 
        {
            Transformed = false;
            TransformTimer = TransformDuration;
        }
        public virtual void Click() { }
        public virtual string OverrideText { get { return "Ability Button"; } }
        public virtual Sprite ButtonSprite { get; }
        public virtual Color32 TextOutlineColor { get { return Color.white; } }
        public void SetCooldown(float cooldown)
        {
            Timer = cooldown;
            Button.SetCoolDown(Timer, Cooldown);
        }
        public void SetNumUses(int numUses)
        {
            CurrentNumUses = numUses;
            Button.SetUsesRemaining(numUses);
        }
        public void SetTransformDuration(float newDuration)
        {
            TransformTimer = newDuration;
            Button.SetCoolDown(Timer, TransformDuration);
        }
        public virtual void MeetingStart(MeetingHud meetingHud)
        {
            if (Transformed)
            {
                Destransform();
            }
        }
        public virtual void Update()
        {
            Color color = Palette.DisabledClear;
            int num = 1;
            bool flag = true;
            if (HaveUses)
            {
                flag = CurrentNumUses > 0;
            }

            if (flag && CanUse && !Minigame.Instance && !MeetingHud.Instance && Vent.currentVent == null)
            {
                color = Palette.EnabledColor;
                num = 0;
            }
            Button.graphic.color = color;
            Button.graphic.material.SetFloat("_Desat", num);
            Button.usesRemainingSprite.color = color;
            Button.usesRemainingSprite.material.SetFloat("_Desat", num);
            Button.usesRemainingText.color = color;
            Button.usesRemainingText.material.SetFloat("_Desat", num);
            Button.buttonLabelText.color = color;
            if (!Transformed)
            {
                if (!MeetingHud.Instance && !ExileController.Instance && Timer > 0f)
                {
                    Timer -= Time.deltaTime;
                    Button.SetCoolDown(Timer, Cooldown);
                }
            }
            else if (!MeetingHud.Instance && !ExileController.Instance && TransformTimer > 0f)
            {
                TransformTimer -= Time.deltaTime;
                Button.SetFillUp(TransformTimer, TransformDuration);
                if (TransformTimer <= 0f)
                {
                    TransformTimer = TransformDuration;
                    Transformed = false;
                    Destransform();
                }
            }
        }
        public virtual void Start() { }
        public virtual void Destroy()
        {
           if (Button != null)
            {
                UnityEngine.Object.Destroy(Button.gameObject);
                Button = null;
            }
        }
        public virtual void Reset(bool creating = false)
        {
            Timer = !creating ? Cooldown : TutorialManager.InstanceExists ? 0f : InitialCooldown;
            CurrentNumUses = NumUses;
            Transformed = false;
            TransformTimer = TransformDuration;
        }
        public AbilityButton CreateButton()
        {
            if (Button != null)
            {
                Destroy();
            }
            Reset(true);
            Button = UnityEngine.Object.Instantiate(HudManager.Instance.AbilityButton, Location == ButtonLocation.BottomRight ? HudHelper.BottomRight : HudHelper.BottomLeft);
            Button.name = GetType().Name;
            PassiveButton component = Button.GetComponent<PassiveButton>();
            Button.graphic.sprite = ButtonSprite;
            Button.graphic.color = new Color(1f, 1f, 1f, 1f);
            Button.gameObject.SetActive(true);
            Button.OverrideText(OverrideText);
            Button.buttonLabelText.SetOutlineColor(TextOutlineColor);
            Button.GetComponent<PassiveButton>().SetNewAction(delegate
            {
                if (CanClick && CanUse)
                {
                    bool flag = true;
                    bool flag2 = true;
                    if (TransformButton)
                    {
                        flag = !Transformed;
                    }

                    if (HaveUses)
                    {
                        flag2 = CurrentNumUses > 0;
                    }

                    if (flag && flag2 && Timer <= 0f)
                    {
                        Timer = Cooldown;
                        if (HaveUses)
                        {
                            CurrentNumUses--;
                            if (Button != null)
                            {
                                Button.SetUsesRemaining(CurrentNumUses);
                            }
                        }
                        Click();
                        if (TransformButton)
                        {
                            Transformed = true;
                        }
                    }

                    if (!flag)
                    {
                        Transformed = false;
                        TransformTimer = TransformDuration;
                        Destransform();
                    }
                }
            });
            Button.SetCoolDown(Timer, Cooldown);
            if (HaveUses)
            {
                Button.SetUsesRemaining(NumUses);
            }
            else
            {
                Button.usesRemainingSprite.gameObject.SetActive(false);
            }
            return Button;
        }
    }
}
// You have some buttons bases to help
using FungleAPI.Attributes;
using FungleAPI.Hud;

namespace FungleAPI.Base.Buttons
{
    [FungleIgnore]
    public class RoleButton<T> : CustomAbilityButton where T : RoleBehaviour
    {
        public PlayerControl Player => PlayerControl.LocalPlayer;
        public RoleBehaviour Role => Player != null && Player.Data != null ? Player.Data.Role : null;
        public override bool Active => Role != null && Role.GetType() == typeof(T);
    }
}
using FungleAPI.Attributes;

namespace FungleAPI.Base.Buttons
{
    [FungleIgnore]
    public class RoleTargetButton<TTarget, TRole> : TargetButton<TTarget> where TTarget : UnityEngine.Object where TRole : RoleBehaviour
    {
        public PlayerControl Player => PlayerControl.LocalPlayer;
        public RoleBehaviour Role => Player != null && Player.Data != null ? Player.Data.Role : null;
        public override bool Active => Role != null && Role.GetType() == typeof(TRole);
        public override bool CanUse => Target != null;
        public override bool CanClick => CanUse;
    }
}
using FungleAPI.Attributes;
using FungleAPI.Hud;
using Rewired.Utils;

namespace FungleAPI.Base.Buttons
{
    [FungleIgnore]
    public class TargetButton<T> : CustomAbilityButton where T : UnityEngine.Object
    {
        public T Target;
        public override bool CanUse => Target != null;
        public override bool CanClick => CanUse;
        public virtual void SetOutline(T target, bool active)
        {
        }
        public virtual T GetTarget()
        {
            return default;
        }
        public override void Update()
        {
            base.Update();
            T newTarget = GetTarget();
            if (newTarget != Target)
            {
                if (Target != null && !Target.IsNullOrDestroyed())
                {
                    SetOutline(Target, false);
                }
                if (newTarget != null && !newTarget.IsNullOrDestroyed())
                {
                    SetOutline(newTarget, true);
                }
                Target = newTarget;
            }
        }
    }
}
// How to create a button
using FungleAPI.Hud;
using UnityEngine;

public class ModButton : CustomAbilityButton
{
    public override bool Active => true;
    public override bool CanClick => true;
    public override bool CanUse => true;
    public override Sprite ButtonSprite => null;
    public override string OverrideText => "Button";
    public override void Click()
    {
        Debug.Log("Button Clicked!");
    }
}
# Translator
// Translator is a easy way to create StringNames and Translations
// Translator class
using System.Collections.Generic;

namespace FungleAPI.Translation
{
    public class Translator
    {
        public Translator(string defaultText)
        {
            Default = defaultText;
            StringName = (StringNames)validId;
            validId--;
            All.Add(this);
        }
        internal static int validId = -99999;
        internal static List<Translator> All = new List<Translator>();
        internal string Default;
        public StringNames StringName;
        internal Dictionary<SupportedLangs, string> Strings = new Dictionary<SupportedLangs, string>();
        public static Translator None = new Translator("STRMISS");
        public Translator AddTranslation(SupportedLangs lang, string text)
        {
            if (!Strings.ContainsKey(lang))
            {
                Strings.Add(lang, text);
            }
            return this;
        }
        public string GetString()
        {
            foreach (KeyValuePair<SupportedLangs, string> pair in Strings)
            {
                if (pair.Key == TranslationController.Instance.currentLanguage.languageID)
                {
                    return pair.Value;
                }
            }
            return Default;
        }
        public override string ToString()
        {
            return GetString();
        }
    }
}
# Configurations
// Classes
using FungleAPI.PluginLoading;
using FungleAPI.Translation;
using FungleAPI.Utilities;
using FungleAPI.Utilities.Prefabs;
using HarmonyLib;
using System;
using System.Collections.Generic;
using System.Reflection;
using UnityEngine;

namespace FungleAPI.Configuration.Attributes
{
    [HarmonyPatch(typeof(StringOption))]
    [AttributeUsage(AttributeTargets.Property, AllowMultiple = false)]
    public class ModdedEnumOption : ModdedOption
    {
        public ModdedEnumOption(string configName, string defaultValue)
            : base(configName)
        {
            Data = ScriptableObject.CreateInstance<StringGameSetting>().DontUnload();
            StringGameSetting stringGameSetting = (StringGameSetting)Data;
            stringGameSetting.Title = ConfigName.StringName;
            stringGameSetting.Type = OptionTypes.String;
            Values = defaultValue.Split("|");
            List<StringNames> stringNames = new List<StringNames>();
            foreach (string str in Values)
            {
                stringNames.Add(new Translator(str).StringName);
            }
            stringGameSetting.Values = stringNames.ToArray();
        }
        public override void Initialize(PropertyInfo property)
        {
            base.Initialize(property);
            if (property.PropertyType == typeof(int) || property.PropertyType == typeof(string))
            {
                ModPlugin plugin = ModPluginManager.GetModPlugin(property.DeclaringType.Assembly);
                int value = property.PropertyType == typeof(int) ? (int)property.GetValue(null) : Values.GetIndex((string)property.GetValue(null));
                localValue = plugin.BasePlugin.Config.Bind(plugin.ModName + " - " + property.DeclaringType.FullName, ConfigName.Default, value.ToString());
                onlineValue = value.ToString();
                FullConfigName = plugin.ModName + property.DeclaringType.FullName + property.Name + value.GetType().FullName;
                Data.SafeCast<StringGameSetting>().Index = int.Parse(localValue.Value);
            }
        }
        public override OptionBehaviour CreateOption(Transform transform)
        {
            StringOption stringOption = UnityEngine.Object.Instantiate(PrefabUtils.Prefab<StringOption>(), transform);
            StringGameSetting stringGameSetting = Data as StringGameSetting;
            stringOption.SetUpFromData(Data, 20);
            stringOption.OnValueChanged = new Action<OptionBehaviour>(delegate
            {
                stringGameSetting.Index = stringOption.Value;
            });
            stringOption.Title = stringGameSetting.Title;
            stringOption.Values = stringGameSetting.Values;
            stringOption.Value = stringGameSetting.Index;
            FixOption(stringOption);
            return stringOption;
        }
        public override object GetReturnedValue()
        {
            Type type = Property.PropertyType;
            if (Property.PropertyType == typeof(int) || Property.PropertyType == typeof(string))
            {
                return Property.PropertyType == typeof(int) ? int.Parse(GetValue()) : Values[int.Parse(GetValue())];
            }
            return GetValue();
        }
        public string[] Values;
        [HarmonyPatch("Initialize")]
        [HarmonyPrefix]
        public static bool InitializePrefix(StringOption __instance)
        {
            if (__instance.name == "ModdedOption")
            {
                __instance.TitleText.text = DestroyableSingleton<TranslationController>.Instance.GetString(__instance.Title);
                __instance.ValueText.text = DestroyableSingleton<TranslationController>.Instance.GetString(__instance.Values[__instance.Value]);
                __instance.AdjustButtonsActiveState();
                return false;
            }
            return true;
        }
        [HarmonyPatch("UpdateValue")]
        [HarmonyPrefix]
        public static bool UpdateValuePrefix(StringOption __instance)
        {
            return __instance.name != "ModdedOption";
        }
    }
}
using AmongUs.GameOptions;
using FungleAPI.Utilities;
using System;
using System.Reflection;
using UnityEngine;
using HarmonyLib;
using FungleAPI.Utilities.Prefabs;
using FungleAPI.PluginLoading;

namespace FungleAPI.Configuration.Attributes
{
    [HarmonyPatch(typeof(NumberOption))]
    [AttributeUsage(AttributeTargets.Property, AllowMultiple = false)]
    public class ModdedNumberOption : ModdedOption
    {
        public ModdedNumberOption(string configName, float minValue, float maxValue, float increment = 1, string formatString = null, bool zeroIsInfinity = false, NumberSuffixes suffixType = NumberSuffixes.Seconds)
            : base(configName)
        {
            Data = ScriptableObject.CreateInstance<FloatGameSetting>().DontUnload();
            FloatGameSetting floatGameSetting = (FloatGameSetting)Data;
            floatGameSetting.Type = OptionTypes.Float;
            floatGameSetting.Title = ConfigName.StringName;
            floatGameSetting.Increment = increment;
            floatGameSetting.ValidRange = new FloatRange(minValue, maxValue);
            floatGameSetting.FormatString = formatString;
            floatGameSetting.ZeroIsInfinity = zeroIsInfinity;
            floatGameSetting.SuffixType = suffixType;
            floatGameSetting.OptionName = FloatOptionNames.Invalid;
        }
        public override void Initialize(PropertyInfo property)
        {
            base.Initialize(property);
            if (property.PropertyType == typeof(float) || property.PropertyType == typeof(int))
            {
                ModPlugin plugin = ModPluginManager.GetModPlugin(property.DeclaringType.Assembly);
                float value = float.Parse(property.GetValue(null).ToString());
                localValue = plugin.BasePlugin.Config.Bind(plugin.ModName + " - " + property.DeclaringType.FullName + " - Option", ConfigName.Default, value.ToString());
                onlineValue = value.ToString();
                FullConfigName = plugin.ModName + property.DeclaringType.FullName + property.Name + value.GetType().FullName;
                Data.SafeCast<FloatGameSetting>().Value = float.Parse(localValue.Value);
            }
        }
        public override object GetReturnedValue()
        {
            Type type = Property.PropertyType;
            if (type == typeof(int))
            {
                return int.Parse(GetValue());
            }
            else if (type == typeof(float))
            {
                return float.Parse(GetValue());
            }
            return GetValue();
        }
        public override OptionBehaviour CreateOption(Transform transform)
        {
            NumberOption numberOption = UnityEngine.Object.Instantiate(PrefabUtils.Prefab<NumberOption>(), Vector3.zero, Quaternion.identity, transform);
            FloatGameSetting floatGameSetting = (FloatGameSetting)Data;
            numberOption.SetUpFromData(Data, 20);
            numberOption.OnValueChanged = new Action<OptionBehaviour>(delegate
            {
                SetValue(numberOption.Value);
                floatGameSetting.Value = numberOption.Value;
            });
            numberOption.Title = floatGameSetting.Title;
            numberOption.Value = float.Parse(localValue.Value);
            numberOption.oldValue = numberOption.oldValue;
            numberOption.Increment = floatGameSetting.Increment;
            numberOption.ValidRange = floatGameSetting.ValidRange;
            numberOption.FormatString = floatGameSetting.FormatString;
            numberOption.ZeroIsInfinity = floatGameSetting.ZeroIsInfinity;
            numberOption.SuffixType = floatGameSetting.SuffixType;
            numberOption.floatOptionName = FloatOptionNames.Invalid;
            FixOption(numberOption);
            return numberOption;
        }
        [HarmonyPatch("Initialize")]
        [HarmonyPrefix]
        public static bool InitializePrefix(NumberOption __instance)
        {
            if (__instance.name == "ModdedOption")
            {
                __instance.AdjustButtonsActiveState();
                __instance.TitleText.text = DestroyableSingleton<TranslationController>.Instance.GetString(__instance.Title);
                NumberOption numberOption = __instance.data.SafeCast<NumberOption>();
                if (numberOption != null)
                {
                    __instance.Value = numberOption.Value;
                }
                return false;
            }
            return true;
        }
        [HarmonyPatch("UpdateValue")]
        [HarmonyPrefix]
        public static bool UpdateValuePrefix(NumberOption __instance)
        {
            return __instance.name != "ModdedOption";
        }
    }
}
using FungleAPI.Utilities;
using System;
using System.Reflection;
using System.Text;
using UnityEngine;
using FungleAPI.Utilities.Prefabs;
using HarmonyLib;
using FungleAPI.PluginLoading;

namespace FungleAPI.Configuration.Attributes
{
    [HarmonyPatch(typeof(ToggleOption))]
    [AttributeUsage(AttributeTargets.Property, AllowMultiple = false)]
    public class ModdedToggleOption : ModdedOption
    {
        public ModdedToggleOption(string configName)
            : base(configName) 
        {
            Data = ScriptableObject.CreateInstance<CheckboxGameSetting>().DontUnload();
            CheckboxGameSetting checkboxGameSetting = (CheckboxGameSetting)Data;
            checkboxGameSetting.Title = ConfigName.StringName;
            checkboxGameSetting.Type = OptionTypes.Checkbox;
        }
        public override void Initialize(PropertyInfo property)
        {
            base.Initialize(property);
            if (property.PropertyType == typeof(bool))
            {
                ModPlugin plugin = ModPluginManager.GetModPlugin(property.DeclaringType.Assembly);
                bool value = (bool)property.GetValue(null);
                localValue = plugin.BasePlugin.Config.Bind(plugin.ModName + " - " + property.DeclaringType.FullName, ConfigName.Default, value.ToString());
                onlineValue = value.ToString();
                FullConfigName = plugin.ModName + property.DeclaringType.FullName + property.Name + value.GetType().FullName;
            }
        }
        public override object GetReturnedValue()
        {
            Type type = Property.PropertyType;
            bool value = bool.Parse(GetValue());
            if (type == typeof(bool))
            {
                return value;
            }
            else if (type == typeof(int) || type == typeof(float))
            {
                return value ? 1 : 0;
            }
            return GetValue();
        }
        public override OptionBehaviour CreateOption(Transform transform)
        {
            ToggleOption toggleOption = UnityEngine.Object.Instantiate(PrefabUtils.Prefab<ToggleOption>(), Vector3.zero, Quaternion.identity, transform);
            toggleOption.SetUpFromData(Data, 20);
            toggleOption.Title = Data.Title;
            toggleOption.TitleText.text = Data.Title.GetString();
            toggleOption.oldValue = bool.Parse(localValue.Value);
            toggleOption.CheckMark.enabled = toggleOption.oldValue;
            toggleOption.OnValueChanged = new Action<OptionBehaviour>(delegate
            {
                SetValue(toggleOption.CheckMark.enabled);
            });
            FixOption(toggleOption);
            return toggleOption;
        }
        [HarmonyPatch("Initialize")]
        [HarmonyPrefix]
        public static bool InitializePrefix(ToggleOption __instance)
        {
            if (__instance.name == "ModdedOption")
            {
                __instance.TitleText.text = DestroyableSingleton<TranslationController>.Instance.GetString(__instance.Title);
                return false;
            }
            return true;
        }
        [HarmonyPatch("UpdateValue")]
        [HarmonyPrefix]
        public static bool UpdateValuePrefix(ToggleOption __instance)
        {
            return __instance.name != "ModdedOption";
        }
    }
}
// Use example
[ModdedNumberOption("Option1", 0, 30)]
public static float Option1=> 10;
[ModdedNumberOption("Option2", 0, 30)]
public static int Option2=> 10;
[ModdedToggleOption("Option3")]
public static bool Option3 => true;
// Properties values are changed with HarmonyLib on runtime
// You can use this on ModdedTeams and ICustomRoles
# More
// FungleAPI has a automatic register system where the API register everything that you create
// You can avoid this system using the Attribut [FungleIgnore]
// Every role that you create need to have the RoleBehaviour and ICustomRole to register on this system (warning)
// Decompile the API to understand better how to use
