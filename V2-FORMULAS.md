# BOB_IMS schema

## 1. App.Formulas

Palettes are taken from the mockup's CSS variables, which differ from BUILD-GUIDE §1 and §3.4.
See "Palette conflicts" at the bottom of this file.

```powerfx
// ---- status colours: the mockup renders status as coloured TEXT on shaded rows,
//      not as a filled pill. Fill/Ink are retained only for the superseded screens.
StatusTable =
    Table(
        { Status: "Not Started", Color: ColorValue("#8A8886"), Fill: ColorValue("#D2D0CE"), Ink: ColorValue("#201F1E") },
        { Status: "Paused",      Color: ColorValue("#5A5A5A"), Fill: ColorValue("#5A5A5A"), Ink: Color.White },
        { Status: "In Progress", Color: ColorValue("#1E8E3E"), Fill: ColorValue("#0B6A0B"), Ink: Color.White },
        { Status: "In Review",   Color: ColorValue("#8250DF"), Fill: ColorValue("#8764B8"), Ink: Color.White },
        { Status: "At Risk",     Color: ColorValue("#C19C00"), Fill: ColorValue("#FFB900"), Ink: ColorValue("#201F1E") },
        { Status: "Delayed",     Color: ColorValue("#D35400"), Fill: ColorValue("#CA5010"), Ink: Color.White },
        { Status: "Blocked",     Color: ColorValue("#C0392B"), Fill: ColorValue("#A4262C"), Ink: Color.White },
        { Status: "Complete",    Color: ColorValue("#002050"), Fill: ColorValue("#002050"), Ink: Color.White },
        { Status: "Cancelled",   Color: Color.Black,           Fill: Color.Black,           Ink: Color.White }
    );

// ---- hierarchy colours (mockup --lv-*)
LevelColors =
    Table(
        { Lvl: 1, LevelName: "Section",        Fill: ColorValue("#C74E00") },
        { Lvl: 2, LevelName: "Line of Effort", Fill: ColorValue("#55731A") },
        { Lvl: 3, LevelName: "Epic",           Fill: ColorValue("#9B30D9") },
        { Lvl: 4, LevelName: "Task",           Fill: ColorValue("#1F6FEB") }
    );

// ---- canonical "me" for every identity comparison. User().Email can
//      differ from the GAL Mail in casing or even value (alias vs primary SMTP),
//      and Power Fx text "=" is CASE-SENSITIVE — so every AssignedMail/Member
//      check goes through this lowercased GAL lookup instead. Coalesce falls
//      back to User().Email if the profile call ever fails. Named formulas
//      evaluate lazily and cache — this is not a per-row connector call.
MyMail = Lower(Coalesce(Office365Users.UserProfile(User().Email).Mail, User().Email));

// ---- shell chrome (mockup :root)
Navy      = ColorValue("#0E3E5C");
NavyDeep  = ColorValue("#0A2E45");
Accent    = ColorValue("#1F6FEB");
Ground    = ColorValue("#F4F3F1");
Chrome    = ColorValue("#E8E7E5");
LineGrey  = ColorValue("#D8D6D4");
InkOnLight= ColorValue("#1B1B1B");
InkSoft   = ColorValue("#605E5C");

// ---- required-field marker for the create-form captions (V3.28). Those captions are
//      HTML text controls so the asterisk can be red while the caption stays InkSoft —
//      modern Text has no inline colour. Same red as pnErr/peErr and the Blocked status.
ReqMark   = "<span style='color:#C0392B'>*</span>";

SecTint   = ColorValue("#FAF0D3");
SecEdge   = ColorValue("#E0A800");
LoeTint   = ColorValue("#E9F6CE");
LoeEdge   = ColorValue("#8DBF2E");
RowFill   = ColorValue("#EFEEEC");
HeadFill  = ColorValue("#DCDBD9");
BoxFill   = ColorValue("#F5F4F2");
```

## 2. App.OnStart

Adds `AppraisalText` and `CollabMails` to `colRows` (the mockup filters on appraisal and on
"is it mine", both of which must be answerable from the flattened row).

```powerfx
// ---- 0. UI state FIRST. If a data source
//         errors, OnStart halts at that statement — anything after it never runs.
//         State init has no data dependencies, so it must precede every data call.

Set(
    gblAppColors,
    {
        // Primary Colors
        Primary1: ColorValue("#30475E"),      // Navy Blue
        Primary2: ColorValue("#F05454"),      // Light Red
        Primary3: ColorValue("#222831"),      // Dark Blue
        Primary4: ColorValue("#DDDDDD"),      // Light Gray

        // Accent Colors
        Black: ColorValue("#000000"),
        Cyan: ColorValue("#17A2B8"),
        Green: ColorValue("#28A745"),
        Orange: ColorValue("#FD7E14"),
        Red: ColorValue("#DC3545"),
        Teal: ColorValue("#20C997"),
        White: ColorValue("#FFFFFF"),
        Yellow: ColorValue("#FFC107"),

        // Neutral Colors
        GrayDark: ColorValue("#484644"),
        GrayMediumDark: ColorValue("#8A8886"),
        GrayMedium: ColorValue("#B3b0AD"),
        GrayMediumLight: ColorValue("#D2D0CE"),
        GrayLight: ColorValue("#F3F2F1")
    }
);

ClearCollect(colExpanded, Filter(Table({ Tab: "", Key: "" }), false));
Set(varTab, "all");
Set(varPanelMode, "");
Set(varSelected, Blank());
Set(varErrors, Filter(Table({ Field: "", Bad: false }), Bad));

// ---- 1. role gating. Config list 'BOB_IMS_AppRoles'.
//         compare against MyMail (canonical lowercased GAL Mail, App.Formulas).
//         NOTE: the choice column is referenced by its INTERNAL name `AppRole`; if
//         Power Apps Studio's intellisense offers `Role` instead, use that.
ClearCollect(colAppRoles, ShowColumns('BOB Integrated Master Schedule (IMS) App Roles', Member, Role));
Set(varIsAdmin,
    !IsBlank(LookUp(colAppRoles,
    If(CountRows(Filter('BOB Integrated Master Schedule (IMS) App Roles', Title = "App Admin")) > 0, true, false)
    || Lower(Member.Email) = MyMail  && AppRole.Value = "Admin")));
Set(varCanWrite,
    varIsAdmin
    || If(CountRows(Filter('BOB Integrated Master Schedule (IMS) App Roles', Title = "App Branch Staff")) > 0, true, false)
    || !IsBlank(LookUp(colAppRoles, Lower(Member.Email) = Lower(User().Email)))
    || !IsBlank(LookUp(colAppRoles, Lower(Member.Email) = MyMail)));
Set(varCanWriteET,
    If(CountRows(Filter('BOB Integrated Master Schedule (IMS) App Roles', Title = "App Contributor")) > 0, true, false)
    || !IsBlank(LookUp(colAppRoles, Lower(Member.Email) = Lower(User().Email)))
    || !IsBlank(LookUp(colAppRoles, Lower(Member.Email) = MyMail)));

// ---- 1b. FULL Section term set via Flow 'BOBIMS-GetSections'
//          Taxonomy Choices() pages at ~20 terms, so the picker reads
//          colSections instead — fetched live form the term store REST API.
//          ADD THE FLOW TO THE APP before pasting, or .run errors. IfError
//          degrads to an empty picker + warning instead of halting OnStart.
IfError(
    With( {raw: 'BOBIMS-GetSections'.Run().sectionsjson },
        ClearCollect(colSections,
            ForAll(Table(ParseJSON(raw)) As t,
                { Label: Text(t.Value.Label), TermGuid: Text(t.Value.TermGuid) }))),
    Notify("Section list failed to load — reload the app or contact the admin.",
            NotificationType.Warning));
// ---- 2. single source of truth
ClearCollect(colItems,
    AddColumns(
        'BOB Integrated Master Schedule (IMS)',
        SectionText,  Coalesce(ThisRecord.Section.Label, "Unassigned"),
        LevelText,    ThisRecord.Level.Value,
        StatusText,   Coalesce(ThisRecord.Status.Value, "Not Started")
    )
);

// ---- 3. flatten to rows. Lvl 1 = Section band, 2 = LOE band, 3 = Epic, 4 = Task.
ClearCollect(colRows,
    ForAll(Distinct(colItems, SectionText) As s,
        {
            RowKey: "S|" & s.Value, SectionKey: "S|" & s.Value,
            LOEKey: Blank(), EpicKey: Blank(),
            Lvl: 1, RowTitle: s.Value, ItemId: Blank(),
            StatusText: Blank(), AppraisalText: Blank(),
            AssignedName: Blank(), AssignedMail: Blank(), CollabMails: Blank(),
            RowStart: Blank(), RowDue: Blank(), Roadmap: false,
            SortPath: s.Value
        })
);
Collect(colRows,
    ForAll(Filter(colItems, LevelText = "Line of Effort") As it,
        {
            RowKey: "L|" & it.SectionText & "|" & it.'Task Name',
            SectionKey: "S|" & it.SectionText,
            LOEKey: "L|" & it.SectionText & "|" & it.'Task Name', EpicKey: Blank(),
            Lvl: 2, RowTitle: it.'Task Name', ItemId: it.ID,
            StatusText: it.StatusText, AppraisalText: it.Appraisal.Value,
            AssignedName: it.Assigned.DisplayName, AssignedMail: it.Assigned.Email,
            CollabMails: Concat(it.Collaborators, Email, ";"),
            RowStart: it.Start, RowDue: it.Due, Roadmap: it.'Roadmap Item',
            SortPath: it.SectionText & "|" & it.'Task Name'
        })
);
Collect(colRows,
    ForAll(Filter(colItems, LevelText = "Epic") As it,
        {
            RowKey: "E|" & it.SectionText & "|" & it.'Line of Effort' & "|" & it.'Task Name',
            SectionKey: "S|" & it.SectionText,
            LOEKey: "L|" & it.SectionText & "|" & it.'Line of Effort',
            EpicKey: "E|" & it.SectionText & "|" & it.'Line of Effort' & "|" & it.'Task Name',
            Lvl: 3, RowTitle: it.'Task Name', ItemId: it.ID,
            StatusText: it.StatusText, AppraisalText: it.Appraisal.Value,
            AssignedName: it.Assigned.DisplayName, AssignedMail: it.Assigned.Email,
            CollabMails: Concat(it.Collaborators, Email, ";"),
            RowStart: it.Start, RowDue: it.Due, Roadmap: it.'Roadmap Item',
            SortPath: it.SectionText & "|" & it.'Line of Effort' & "|" & it.'Task Name'
        })
);
Collect(colRows,
    ForAll(Filter(colItems, LevelText = "Task") As it,
        {
            RowKey: "T|" & it.ID,
            SectionKey: "S|" & it.SectionText,
            LOEKey: "L|" & it.SectionText & "|" & it.'Line of Effort',
            EpicKey: "E|" & it.SectionText & "|" & it.'Line of Effort' & "|" & it.Epic,
            Lvl: 4, RowTitle: it.'Task Name', ItemId: it.ID,
            StatusText: it.StatusText, AppraisalText: it.Appraisal.Value,
            AssignedName: it.Assigned.DisplayName, AssignedMail: it.Assigned.Email,
            CollabMails: Concat(it.Collaborators, Email, ";"),
            RowStart: it.Start, RowDue: it.Due, Roadmap: it.'Roadmap Item',
            SortPath: it.SectionText & "|" & it.'Line of Effort' & "|" & it.Epic & "|" & it.'Task Name'
        })
);

// ---- 4. deep link from notification emails (single-screen form).
If(!IsBlank(Param("itemId")),
    Set(varSelected, LookUp('BOB Integrated Master Schedule (IMS)',
                            ID = Value(Param("itemId"))));
    If(!IsBlank(varSelected), Set(varPanelMode, "read")));
```
