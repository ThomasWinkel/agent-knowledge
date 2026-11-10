---
title: Visio text under flips - alignment, margins, upright text by formula
description: Read when a text must keep its side, alignment and margins when its shape or group is flipped or turned (upright text, overflow away from an anchor) - Visio's unmirroring, formulas, why not SETF.
---

# Visio text under flips: alignment, margins, upright text by formula

Visio never draws text mirrored. A flip moves the text block (local coordinates), but the text inside is laid out as seen: a text block aligned left in a flipped shape shows its text on the left **as seen**, i.e. on the shape's other side. Measured with Visio Pro 2019 C2R x64 (build 20527), COM from PowerShell, pictures of exported pages.

## How Visio unmirrors

- An odd count of `FlipX` (own + all parent groups) reverses the **reading direction** inside the text block; an odd count of `FlipY` reverses its **up direction**. `FlipX` + `FlipY`: neither (text not turned).
- So the same page matrix gives different text: `FlipX=1, Angle=90 deg` reads bottom-to-top, `FlipY=1, Angle=-90 deg` (same matrix) reads top-to-bottom. A formula cannot derive the text direction from the page transform alone: count the flip cells (own `FlipX` plus the parent's `Sheet.n!FlipX`; ShapeSheet has no parent reference, so deeper groups are not seen).
- `Selection.FlipHorizontal()` on a turned shape sets `FlipX` and keeps `Angle`; then `FlipVertical()` gives `FlipX 1, FlipY 1, Angle -90 deg`.

## Recipe (pure formulas, Undo-safe)

Intent is Shape Data in the shape's own coordinates (`Prop.Align` = `Left;Center;Right`, `Prop.VAlign` = `Top;Middle;Bottom`, `Prop.Rotation` = `With Shape;Horizontal;Readable`, `Prop.Wrap`); everything else is GUARDed formulas:

```text
User.A      = MAX(0,LOOKUP(Prop.Align,Prop.Align.Format))          0 left, 1 center, 2 right
User.FX     = IF(FlipX<>Sheet.<group>!FlipX,1,0)                     (alone: IF(FlipX,1,0)); User.FY alike
User.Dir    = ATAN2(PNTY(LOCTOPAR(PNT(1,0),EventXFMod,ThePage!EventXFMod))-PNTY(LOCTOPAR(PNT(0,0),EventXFMod,ThePage!EventXFMod)),
                    PNTX(LOCTOPAR(PNT(1,0),EventXFMod,ThePage!EventXFMod))-PNTX(LOCTOPAR(PNT(0,0),EventXFMod,ThePage!EventXFMod)))
read        = PI()-MODULUS(PI()-User.Dir-User.FX*PI(),2*PI())        reading direction on the page at TxtAngle 0
TxtAngle    = GUARD(IF(mode=Horizontal, IF(User.FX=User.FY,-1,1)*(User.Dir+User.FX*PI()),
                    IF(AND(mode=Readable,OR(read>PI()/2+0.0001,read<-PI()/2+0.0001)),PI(),0)))
User.Turned = COS(TxtAngle)<-0.001
User.SwapH  = IF(User.FX<>User.Turned,1,0)        User.SwapV = IF(User.FY<>User.Turned,1,0)
Para.HorzAlign = GUARD(IF(User.SwapH,2-User.A,User.A))                 VerticalAlign alike with SwapV
LeftMargin  = GUARD(IF(User.SwapH,<right>,<left>))                     Right/Top/Bottom alike
TxtWidth    = GUARD(IF(Prop.Wrap,Width,MAX(Width,TEXTWIDTH(TheText))))  TxtHeight = GUARD(Height)
TxtPinX     = GUARD(Width*User.A/2)                                    TxtPinY = GUARD(Height*0.5)
TxtLocPinX  = GUARD(TxtWidth*IF(User.Turned,1-User.A/2,User.A/2))       TxtLocPinY = GUARD(TxtHeight*0.5)
```

Result: an unwrapped text sticks to the aligned side and runs out of the shape on the other side (Center: both); wrapped text grows in height by the vertical alignment (Middle: both ways; Visio overflows vertically by itself). Flipped, the text stays at the same side of the shape, alignment and margins swap as seen. `Readable` = never upside down (read from below or right); `Horizontal` = always horizontal on the page (turns around the anchor). GUARD makes Visio's own Align buttons and margin dialog do nothing on that shape: offer Shape Data instead.

## Gotchas

- **No SETF trigger that swaps alignment/margins on a flip**: it works forward, but Undo of the flip breaks it (the trigger fires again while Undo restores the cells; after Undo the flip is back, alignment and margins stay swapped). Use formulas as above.
- `LOCTOPAGE` and `ANGLETOPAGE` do not exist (`Unknown function name.`); `LOCTOPAR(pt,EventXFMod,ThePage!EventXFMod)` maps to the page. In a sub-shape its value lags one change after a direct cell write on the parent group (`FormulaU`); after `Selection.Flip*`/`Rotate90` it is current.
- `TEXTWIDTH(TheText)` includes the margins and follows margin, font size and text changes in a visible Visio; in an invisible Visio (`InvisibleApp`) it kept the old value.
- `SETF` writes at once: `SETF(GetRef(a),b)+SETF(GetRef(b),a)` sets both to `b`. A swap without a temp cell: `SETF(GetRef(a),a+b)+SETF(GetRef(b),a-b)+SETF(GetRef(a),a-b)`. A trigger cell that reads the cells it writes runs again (up to Visio's circular limit) unless it compares against a stored state.
- `Para.HorzAlign` formulas only reach paragraph row 1; Visio adds rows only for paragraphs formatted differently.
