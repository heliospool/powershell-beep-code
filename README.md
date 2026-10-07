Literally just copy/paste this into PowerShell and hit ENTER:

```powershell
$speed = 1.0   # 1.25 = faster, 0.8 = slower

if (-not ('Chip' -as [type])) {
Add-Type -TypeDefinition @'
using System; using System.IO; using System.Media;
public static class Chip {
    const int Rate = 44100;
    static float[] Buf;
    public static void Init(int totalMs) { Buf = new float[(long)Rate * totalMs / 1000 + Rate]; }
    public static void Voice(double[] f, int[] ms, double speed, double vol, int wave) {
        long pos = 0;
        for (int n = 0; n < f.Length; n++) {
            int len = (int)(Rate * ms[n] / 1000.0 / speed);
            if (f[n] > 0) {
                int sound = len - Math.Min(len / 6, Rate / 50);
                double ph = 0, step = f[n] / Rate;
                for (int i = 0; i < sound && pos + i < Buf.Length; i++) {
                    double v = wave == 0 ? (ph < 0.5 ? 1 : -1) : 4 * Math.Abs(ph - 0.5) - 1;
                    double env = Math.Min(1, i / (Rate * 0.004))
                               * Math.Min(1, (sound - i) / (Rate * 0.008))
                               * (0.55 + 0.45 * Math.Exp(-i / (Rate * 0.12)));
                    Buf[pos + i] += (float)(v * env * vol);
                    ph += step; if (ph >= 1) ph -= 1;
                }
            }
            pos += len;
        }
    }
    public static void Play() {
        var s = new MemoryStream(); var w = new BinaryWriter(s);
        int n = Buf.Length;
        w.Write(0x46464952); w.Write(36 + n * 2); w.Write(0x45564157);
        w.Write(0x20746D66); w.Write(16); w.Write((short)1); w.Write((short)1);
        w.Write(Rate); w.Write(Rate * 2); w.Write((short)2); w.Write((short)16);
        w.Write(0x61746164); w.Write(n * 2);
        foreach (float x in Buf) w.Write((short)(Math.Max(-1.0, Math.Min(1.0, x)) * 32000));
        s.Position = 0;
        new SoundPlayer(s).PlaySync();
    }
}
'@
}

$semi = @{ C = -9; D = -7; E = -5; F = -4; G = -2; A = 0; B = 2 }
function Freq($n) {
    if ($n -eq 'R') { return 0.0 }
    $m = [regex]::Match($n, '^([A-G])(#?)(\d)$')
    $s = $semi[$m.Groups[1].Value] + [int]($m.Groups[2].Value -eq '#') + 12 * ([int]$m.Groups[3].Value - 4)
    440 * [math]::Pow(2, $s / 12)
}
function Parse($text) {
    $f = @(); $d = @()
    foreach ($t in ($text -split '\s+' | Where-Object { $_ })) {
        $n, $ms = $t -split ':'
        $f += [double](Freq $n); $d += [int]$ms
    }
    [pscustomobject]@{ F = [double[]]$f; D = [int[]]$d }
}
function Bass($roots) {
    (($roots -split '\s+' | Where-Object { $_ }) | ForEach-Object { "${_}3:250 ${_}4:250 ${_}3:250 ${_}4:250" }) -join ' '
}

$melA = 'E5:500 B4:250 C5:250 D5:250 E5:125 D5:125 C5:250 B4:250 A4:500 A4:250 C5:250 E5:500 D5:250 C5:250 B4:750 C5:250 D5:500 E5:500 C5:500 A4:500 A4:500 R:500 R:250 D5:500 F5:250 A5:500 G5:250 F5:250 E5:750 C5:250 E5:500 D5:250 C5:250 B4:500 B4:250 C5:250 D5:500 E5:500 C5:500 A4:500 A4:500 R:500'
$melB = 'E5:1000 C5:1000 D5:1000 B4:1000 C5:1000 A4:1000 G#4:1000 B4:1000 E5:1000 C5:1000 D5:1000 B4:1000 C5:500 E5:500 A5:1000 G#5:2000'

$bassA = Bass 'E E A A G# E A A D D C C B E A A'
$bassB = Bass 'A A E E A A E E A A E E A A E E'

$mel  = Parse "$melA $melA $melB $melB"
$bass = Parse "$bassA $bassA $bassB $bassB"

[Chip]::Init([int](($mel.D | Measure-Object -Sum).Sum / $speed))
[Chip]::Voice($mel.F,  $mel.D,  $speed, 0.22, 0)
[Chip]::Voice($bass.F, $bass.D, $speed, 0.35, 1)
[Chip]::Play()
```
