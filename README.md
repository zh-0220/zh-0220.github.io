# 國立中正大學115學年度特選才-汪子恒加班費計算系統
<html>
import React, { useMemo, useState, useEffect } from "react";
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { Trash2, PlusCircle, Download, Info } from "lucide-react";

// --- Helpers ---
const currency = (n:number) => n.toLocaleString("zh-TW", { style: "currency", currency: "TWD", maximumFractionDigits: 0 });
const num = (v: string | number, def=0) => {
  const n = typeof v === 'string' ? parseFloat(v) : v;
  return isFinite(n) ? n : def;
}

// Taiwan MoL common conversion: 時薪 = 月薪 ÷ 240 (30日 * 8小時)
const monthlyToHourly = (m:number) => m / 240;

// Types
type DayType = "workday" | "restday" | "holiday";

type Row = {
  id: string;
  date: string; // YYYY-MM-DD
  type: DayType;
  hours: number; // total on that day
  note?: string;
};

const defaultRows: Row[] = [];

const defaultSettings = {
  mode: "hourly" as "hourly" | "monthly",
  hourlyWage: 200,
  monthlyWage: 42000,
  // Multipliers (editable)
  workdayFirst2x: 1.34,
  workdayNext2x: 1.67,
  // Rest day blocks use same multipliers but with minimum paid blocks (2/4/8h)
  restFirst2x: 1.34,
  restNext6x: 1.67,
  // Holiday (customizable; practices vary)
  holidayAllx: 2.0,
  // Limits
  capWorkdayHours: 4, // 法規通常每天延長工時上限4小時
  capRestHolidayHours: 8, // 常見實務上限 8h
  useRestBlocks: true,
}

function useLocalStorage<T>(key:string, init:T){
  const [state,setState] = useState<T>(()=>{
    try{ const s = localStorage.getItem(key); return s? JSON.parse(s) as T : init; }catch{ return init }
  });
  useEffect(()=>{ try{ localStorage.setItem(key, JSON.stringify(state)); }catch{} },[key,state]);
  return [state,setState] as const;
}

export default function OvertimePayApp(){
  const [settings, setSettings] = useLocalStorage("ot.settings", defaultSettings);
  const [rows, setRows] = useLocalStorage<Row[]>("ot.rows", defaultRows);

  const hourly = settings.mode === 'hourly' ? settings.hourlyWage : monthlyToHourly(settings.monthlyWage);

  const addRow = () => {
    setRows(r=>[...r,{ id: crypto.randomUUID(), date: new Date().toISOString().slice(0,10), type: "workday", hours: 1 }]);
  }
  const deleteRow = (id:string)=> setRows(r=> r.filter(x=> x.id!==id));
  const updateRow = (id:string, patch: Partial<Row>) => setRows(r=> r.map(x=> x.id===id? {...x, ...patch}: x));
  const clearAll = ()=> setRows([]);

  // Calculation logic
  const calcRow = (row: Row) => {
    const h = Math.max(0, row.hours);
    const hourlyWage = hourly;

    if(row.type === 'workday'){
      const cap = settings.capWorkdayHours;
      const effectiveH = Math.min(h, cap);
      const first2 = Math.min(2, effectiveH);
      const next2 = Math.min(2, Math.max(0, effectiveH - 2));
      const extra = Math.max(0, h - cap);
      const pay = first2 * hourlyWage * settings.workdayFirst2x + next2 * hourlyWage * settings.workdayNext2x;
      return { pay, detail: { first2, next2, extra }, warning: extra>0 ? `已超過工作日加班上限 ${cap} 小時，僅計算上限內工時。` : "" };
    }

    if(row.type === 'restday'){
      const cap = settings.capRestHolidayHours;
      const effectiveH = Math.min(h, cap);
      let billableFirst = 0, billableNext = 0, warning = "";

      if(settings.useRestBlocks){
        // Block minima: <=2h → 2h(1.34x); >2 & <=4 → 4h(2h@1.34 + 2h@1.67); >4 & <=8 → 8h(2h@1.34 + 6h@1.67)
        if(effectiveH <= 2){ billableFirst = 2; billableNext = 0; }
        else if(effectiveH <= 4){ billableFirst = 2; billableNext = 2; }
        else { billableFirst = 2; billableNext = Math.min(6, effectiveH - 2); }
      }else{
        billableFirst = Math.min(2, effectiveH);
        billableNext = Math.min(6, Math.max(0, effectiveH - 2));
      }
      const extra = Math.max(0, h - cap);
      const pay = billableFirst * hourlyWage * settings.restFirst2x + billableNext * hourlyWage * settings.restNext6x;
      warning = extra>0 ? `已超過休息日計算上限 ${cap} 小時，僅計算上限內工時。` : warning;
      return { pay, detail: { first2: billableFirst, next6: billableNext, extra }, warning };
    }

    // holiday
    {
      const cap = settings.capRestHolidayHours;
      const effectiveH = Math.min(h, cap);
      const extra = Math.max(0, h - cap);
      const pay = effectiveH * hourlyWage * settings.holidayAllx;
      const warning = extra>0 ? `已超過假日計算上限 ${cap} 小時，僅計算上限內工時。` : "";
      return { pay, detail: { billable: effectiveH, extra }, warning };
    }
  }

  const summary = useMemo(()=>{
    let total = 0;
    const lines = rows.map(r=>{
      const { pay } = calcRow(r);
      total += pay;
      return { id:r.id, date:r.date, type:r.type, hours:r.hours, pay };
    });
    return { total, lines };
  },[rows, settings, hourly]);

  const exportCSV = () => {
    const headers = ["日期","類型","加班時數","加班費(TWD)"];
    const mapType: Record<DayType,string> = { workday:"工作日(延長工時)", restday:"休息日", holiday:"國定假日/特休日" };
    const rowsCsv = summary.lines.map(l=> [l.date, mapType[l.type], l.hours, Math.round(l.pay)].join(","));
    const text = [headers.join(","), ...rowsCsv, "合計,,", Math.round(summary.total)].join("\n");
    const blob = new Blob(["\uFEFF"+text], { type: "text/csv;charset=utf-8;" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = `加班費明細_${new Date().toISOString().slice(0,10)}.csv`;
    a.click();
  }

  return (
    <div className="min-h-screen bg-gray-50 p-4 md:p-8">
      <div className="mx-auto max-w-6xl space-y-6">
        <header className="flex items-center justify-between">
          <h1 className="text-2xl md:text-3xl font-bold tracking-tight">加班費計算系統（臺灣預設）</h1>
          <div className="text-sm text-gray-500 flex items-center gap-2"><Info size={16}/> 本工具僅供估算，實際給付依公司規章與法規為準。</div>
        </header>

        {/* Settings Card */}
        <Card className="rounded-2xl shadow-sm">
          <CardContent className="p-4 md:p-6">
            <Tabs defaultValue="wage" className="w-full">
              <TabsList className="grid grid-cols-2 w-full max-w-md">
                <TabsTrigger value="wage">基本資料</TabsTrigger>
                <TabsTrigger value="rules">計算規則</TabsTrigger>
              </TabsList>

              <TabsContent value="wage" className="pt-4">
                <div className="grid md:grid-cols-3 gap-4">
                  <div className="space-y-2">
                    <Label>薪資模式</Label>
                    <Select value={settings.mode} onValueChange={(v)=>setSettings(s=>({...s, mode: v as any}))}>
                      <SelectTrigger><SelectValue /></SelectTrigger>
                      <SelectContent>
                        <SelectItem value="hourly">以「時薪」計算</SelectItem>
                        <SelectItem value="monthly">以「月薪」推算時薪</SelectItem>
                      </SelectContent>
                    </Select>
                  </div>
                  {settings.mode === 'hourly' ? (
                    <div className="space-y-2">
                      <Label>時薪（TWD）</Label>
                      <Input type="number" min={0} value={settings.hourlyWage}
                        onChange={(e)=>setSettings(s=>({...s, hourlyWage: num(e.target.value, s.hourlyWage)}))}/>
                    </div>
                  ) : (
                    <>
                      <div className="space-y-2">
                        <Label>月薪（TWD）</Label>
                        <Input type="number" min={0} value={settings.monthlyWage}
                          onChange={(e)=>setSettings(s=>({...s, monthlyWage: num(e.target.value, s.monthlyWage)}))}/>
                        <p className="text-xs text-gray-500">換算時薪 = 月薪 ÷ 240 = {Math.round(monthlyToHourly(settings.monthlyWage))} 元</p>
                      </div>
                      <div className="space-y-2">
                        <Label className="whitespace-nowrap">換算時薪（自動）</Label>
                        <Input readOnly value={Math.round(hourly)} />
                      </div>
                    </>
                  )}
                </div>
              </TabsContent>

              <TabsContent value="rules" className="pt-4">
                <div className="grid md:grid-cols-3 gap-4">
                  <div className="space-y-2">
                    <Label>工作日：前2小時倍率</Label>
                    <Input type="number" step="0.01" value={settings.workdayFirst2x}
                      onChange={(e)=>setSettings(s=>({...s, workdayFirst2x: num(e.target.value, s.workdayFirst2x)}))}/>
                    <p className="text-xs text-gray-500">常見：1.34 倍</p>
                  </div>
                  <div className="space-y-2">
                    <Label>工作日：第3~4小時倍率</Label>
                    <Input type="number" step="0.01" value={settings.workdayNext2x}
                      onChange={(e)=>setSettings(s=>({...s, workdayNext2x: num(e.target.value, s.workdayNext2x)}))}/>
                    <p className="text-xs text-gray-500">常見：1.67 倍（超過4小時不計；將給出警示）</p>
                  </div>
                  <div className="space-y-2">
                    <Label>休息日：前2小時倍率</Label>
                    <Input type="number" step="0.01" value={settings.restFirst2x}
                      onChange={(e)=>setSettings(s=>({...s, restFirst2x: num(e.target.value, s.restFirst2x)}))}/>
                    <p className="text-xs text-gray-500">常見：1.34 倍，並採 2/4/8 小時計費門檻</p>
                  </div>
                  <div className="space-y-2">
                    <Label>休息日：後續6小時倍率</Label>
                    <Input type="number" step="0.01" value={settings.restNext6x}
                      onChange={(e)=>setSettings(s=>({...s, restNext6x: num(e.target.value, s.restNext6x)}))}/>
                    <p className="text-xs text-gray-500">常見：1.67 倍，最多計到 8 小時</p>
                  </div>
                  <div className="space-y-2">
                    <Label>國定假日等：倍率（全時段）</Label>
                    <Input type="number" step="0.01" value={settings.holidayAllx}
                      onChange={(e)=>setSettings(s=>({...s, holidayAllx: num(e.target.value, s.holidayAllx)}))}/>
                    <p className="text-xs text-gray-500">預設 2.00 倍（各單位規則可能不同，可自行調整）</p>
                  </div>
                  <div className="space-y-2">
                    <Label>上限與規則</Label>
                    <div className="grid grid-cols-2 gap-2">
                      <div>
                        <Label className="text-xs">工作日每日上限(小時)</Label>
                        <Input type="number" value={settings.capWorkdayHours}
                          onChange={(e)=>setSettings(s=>({...s, capWorkdayHours: Math.max(0, Math.floor(num(e.target.value, s.capWorkdayHours)))}))}/>
                      </div>
                      <div>
                        <Label className="text-xs">休/假日每日上限(小時)</Label>
                        <Input type="number" value={settings.capRestHolidayHours}
                          onChange={(e)=>setSettings(s=>({...s, capRestHolidayHours: Math.max(0, Math.floor(num(e.target.value, s.capRestHolidayHours)))}))}/>
                      </div>
                    </div>
                    <div className="flex items-center gap-2 mt-2">
                      <input id="restBlocks" type="checkbox" className="h-4 w-4" checked={settings.useRestBlocks}
                        onChange={(e)=>setSettings(s=>({...s, useRestBlocks: e.target.checked}))}/>
                      <Label htmlFor="restBlocks">休息日採 2/4/8 小時最低計費門檻</Label>
                    </div>
                  </div>
                </div>
              </TabsContent>
            </Tabs>
          </CardContent>
        </Card>

        {/* Entry Card */}
        <Card className="rounded-2xl shadow-sm">
          <CardContent className="p-4 md:p-6 space-y-4">
            <div className="flex items-center justify-between">
              <h2 className="font-semibold text-lg">本期明細</h2>
              <div className="flex gap-2">
                <Button variant="secondary" onClick={addRow}><PlusCircle className="mr-2 h-4 w-4"/>新增一天</Button>
                <Button variant="outline" onClick={clearAll}><Trash2 className="mr-2 h-4 w-4"/>清空</Button>
                <Button onClick={exportCSV}><Download className="mr-2 h-4 w-4"/>匯出 CSV</Button>
              </div>
            </div>

            <div className="overflow-x-auto">
              <table className="w-full text-sm">
                <thead>
                  <tr className="text-left text-gray-500">
                    <th className="py-2">日期</th>
                    <th className="py-2">類型</th>
                    <th className="py-2">時數</th>
                    <th className="py-2">小計</th>
                    <th className="py-2">備註/警示</th>
                    <th></th>
                  </tr>
                </thead>
                <tbody>
                  {rows.length===0 && (
                    <tr><td colSpan={6} className="py-8 text-center text-gray-400">尚無資料，點「新增一天」開始填寫</td></tr>
                  )}
                  {rows.map(r=>{
                    const result = calcRow(r);
                    return (
                      <tr key={r.id} className="border-t">
                        <td className="py-2">
                          <Input type="date" value={r.date} onChange={(e)=>updateRow(r.id,{date:e.target.value})}/>
                        </td>
                        <td className="py-2">
                          <Select value={r.type} onValueChange={(v)=>updateRow(r.id,{type: v as DayType})}>
                            <SelectTrigger className="w-44"><SelectValue /></SelectTrigger>
                            <SelectContent>
                              <SelectItem value="workday">工作日（延長工時）</SelectItem>
                              <SelectItem value="restday">休息日</SelectItem>
                              <SelectItem value="holiday">國定假日/特休日</SelectItem>
                            </SelectContent>
                          </Select>
                        </td>
                        <td className="py-2 w-36">
                          <Input type="number" min={0} step={0.5} value={r.hours} onChange={(e)=>updateRow(r.id,{hours: num(e.target.value, r.hours)})}/>
                        </td>
                        <td className="py-2 font-semibold">{currency(Math.round(result.pay))}</td>
                        <td className="py-2 text-xs text-gray-600">
                          {result.warning && <div className="text-red-600 mb-1">{result.warning}</div>}
                          {r.type==='workday' && <div>計算：前2h × {settings.workdayFirst2x}、第3~4h × {settings.workdayNext2x}</div>}
                          {r.type==='restday' && <div>計算：2h × {settings.restFirst2x}、最多再6h × {settings.restNext6x}{settings.useRestBlocks?"（含2/4/8小時計費門檻）":""}</div>}
                          {r.type==='holiday' && <div>計算：全時段 × {settings.holidayAllx}</div>}
                        </td>
                        <td className="py-2 text-right">
                          <Button variant="ghost" onClick={()=>deleteRow(r.id)}><Trash2 className="h-4 w-4"/></Button>
                        </td>
                      </tr>
                    )
                  })}
                </tbody>
              </table>
            </div>

            <div className="flex items-center justify-between pt-2">
              <div className="text-sm text-gray-500">目前換算時薪：<span className="font-semibold text-gray-700">{Math.round(hourly)} 元</span></div>
              <div className="text-lg md:text-xl font-bold">本期加班費合計：<span className="text-emerald-600">{currency(Math.round(summary.total))}</span></div>
            </div>
          </CardContent>
        </Card>

        {/* Notes */}
        <Card className="rounded-2xl shadow-sm">
          <CardContent className="p-4 md:p-6 text-sm text-gray-600 space-y-2">
            <p className="font-semibold">說明與假設</p>
            <ul className="list-disc pl-5 space-y-1">
              <li>月薪換算時薪採 <span className="font-mono">月薪 ÷ 240</span>（30 天 × 8 小時）之常見作法。</li>
              <li>工作日延長工時：預設前 2 小時 1.34 倍、再 2 小時 1.67 倍，並以每日 4 小時為上限。</li>
              <li>休息日預設採 2/4/8 小時最低計費門檻（可關閉），前 2 小時 1.34 倍，其後最多 6 小時 1.67 倍。</li>
              <li>國定假日/特休日倍率差異較大，預設 2.00 倍，請依單位規章或勞動契約自行調整。</li>
              <li>法規與內規可能隨時間調整，結果僅供估算，請與公司人資或勞工局規定比對確認。</li>
            </ul>
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
<html>
