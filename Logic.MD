# This MD describes the underlying logic in a simple way, using pseudocode, the tekken related stuff (like the key bit) may change each build
```cpp
enum class MoveLevel : int32_t
{
    UNKNOWN = -1,
    LOW = 0,
    MID,
    HIGH,
    SPECIALMID
    //im missing the special low, whatever
};

struct Requirement
{
    uint32_t Id;
    uint32_t Parameter;
};

struct EscapeExtra
{
    uint32_t Marker;
};

struct CancelData
{
    uint64_t Command;
    Requirement* Requirements;
    EscapeExtra* Extra;
    uint32_t FirstFrame;
    uint32_t LastFrame;
};

struct MoveData
{
    uint64_t EncodedFlags[2];
    CancelData* Cancels;
    int32_t ActiveStart;
    int32_t ActiveEnd;
};

struct Character
{
    MoveData* Move;
    int32_t Frame;
    uint8_t CounterHit;
    uint32_t ThrowPunish;
    uint32_t State;
};
```

```cpp
MoveLevel DetectAttack(Character* Fighter)
{
    constexpr uint32_t FlagsMask = 0xFFFFFFE0u;

    auto Move = Read<MoveData*>(Fighter->Move);

    uint64_t First  = Read<uint64_t>(Move->EncodedFlags[0]);
    uint64_t Second = Read<uint64_t>(Move->EncodedFlags[1]);

    uint32_t Low = static_cast<uint32_t>(First);
    uint32_t Shifted = Low;
    uint32_t Checksum = 0;

    for (int Rotation = 8; Rotation <= 32; Rotation += 8)
    {
        Checksum ^= static_cast<uint32_t>(std::rotl(Second, Rotation)) ^ Shifted;
        Shifted >>= 8;
    }

    if (Checksum == 0)
        Checksum = 1;

    if (static_cast<uint32_t>(First >> 32) != Checksum)
        return MoveLevel::UNKNOWN;

    uint32_t Key = Low ^ 0x1D;
    uint32_t Rotated = static_cast<uint32_t>(std::rotl(Second, static_cast<int>(Key & 31)));
    uint32_t Flags = (Rotated & FlagsMask) ^ Key;

    if (Read<MoveData*>(Fighter->Move) != Move)
        return MoveLevel::UNKNOWN;

    uint32_t Level = Flags & 0xFFFF;

    if (Level == 0x0512)
        return MoveLevel::HIGH;
    else if (Level == 0x0217)
        return MoveLevel::MID;
    else if (Level == 0x010F)
        return MoveLevel::LOW;
    else if (Level == 0x041F)
        return MoveLevel::SPECIALMID;
    else
        return MoveLevel::UNKNOWN;
}
```

```cpp
auto Frame = Read<int32_t>(Fighter->Frame);
auto Start = Read<int32_t>(Move->ActiveStart);
auto End   = Read<int32_t>(Move->ActiveEnd);

bool ActiveNow = false;

if (Start >= 0 && End >= Start && Frame >= Start && Frame <= End)
{
    ActiveNow = true;
}
```

```cpp
auto Reaction = Read<MoveData*>(Defender->Move);
auto Cancels = Read<CancelData*>(Reaction->Cancels);
auto Frame = Read<int32_t>(Defender->Frame);
bool CounterHit = Read<uint8_t>(Attacker->CounterHit) != 0;

if (Read<uint32_t>(Defender->ThrowPunish) != 0)
    return;

if ((Read<uint32_t>(Defender->State) & 0x1000) != 0)
    return;

uint8_t AcceptedButtons = 0;
bool BreakWindowOpen = false;

for (const auto& Cancel : ReadCancelList(Cancels))
{
    if (Cancel.Command != 0 || Cancel.FirstFrame > Cancel.LastFrame)
        continue;

    auto Extra = Read<EscapeExtra*>(Cancel.Extra);
    if (Read<uint32_t>(Extra->Marker) != 6876)
        continue;

    uint8_t PathButtons = 0;
    bool ContextMatches = true;
    bool Supported = true;

    for (const auto& Req : ReadRequirementList(Cancel.Requirements))
    {
        if (Req.Id == 460 && Req.Parameter == 0)
            ContextMatches = CounterHit;
        else if (Req.Id == 461 && Req.Parameter == 0)
            ContextMatches = !CounterHit;

        else if (PathButtons == 0 && Req.Parameter == 0 && Req.Id >= 131 && Req.Id <= 133)
            PathButtons = static_cast<uint8_t>(1u << (Req.Id - 131)); // 131 = 1, 132 = 2, 133 = 1+2

        else if (PathButtons == 0 && Req.Id == 135 && Req.Parameter >= 1 && Req.Parameter <= 3)
            PathButtons = static_cast<uint8_t>(1u << (Req.Parameter - 1)); // Parameter 1 = 1, 2 = 2, 3 = 1+2
        else
            Supported = false;
    }

    if (!Supported || !ContextMatches || PathButtons == 0)
        continue;

    if (Frame >= 0 && static_cast<uint32_t>(Frame) >= Cancel.FirstFrame && static_cast<uint32_t>(Frame) <= Cancel.LastFrame)
    {
        AcceptedButtons |= PathButtons;
        BreakWindowOpen = true;
    }
}

if (AutoThrowBreak && InputReady && BreakWindowOpen)
{
    if (AcceptedButtons & 1) // This is 1.
        Press("1");
    else if (AcceptedButtons & 2) // This is 2.
        Press("2");
    else if (AcceptedButtons & 4) // This is 1+2.
        Press("1+2");
}
```
