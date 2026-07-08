// api/whatsapp.js
// Webhook de Twilio WhatsApp — Ruiz Vitela Bitácora
// Recibe mensajes, consulta Supabase, responde estatus de viajes (solo lectura)

const SB_URL = process.env.SUPABASE_URL || "https://fjiyetlufaxcyxngmxvt.supabase.co";
const SB_KEY = process.env.SUPABASE_ANON_KEY || "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImZqaXlldGx1ZmF4Y3l4bmdteHZ0Iiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODI3NTk3MDksImV4cCI6MjA5ODMzNTcwOX0.Xw60KTCo9P4ZFJtef7H-QPKXeLIchR99YI1olj_U2f4";

// ══ WHITELIST — SOLO estos números pueden usar el bot ══
// Formato Twilio: "whatsapp:+52XXXXXXXXXX"
const ALLOWED_NUMBERS = (process.env.WHATSAPP_ALLOWED_NUMBERS || "")
  .split(",")
  .map(n => n.trim())
  .filter(Boolean);
// Variable de entorno en Vercel:
// WHATSAPP_ALLOWED_NUMBERS=whatsapp:+523312345678,whatsapp:+523398765432

// ══ RUTAS (mismo catálogo que la app) ══
const RUTAS = [
  {id:"S-Sauz",c:"Siderurgica",n:"Siderurgica-Sauz",t:180},
  {id:"S-Chulavista",c:"Siderurgica",n:"Siderurgica-Chulavista",t:180},
  {id:"S-Colina",c:"Siderurgica",n:"Siderurgica-Colina del Roble",t:180},
  {id:"S-Verde",c:"Siderurgica",n:"Siderurgica-El Verde",t:180},
  {id:"S-Loma",c:"Siderurgica",n:"Siderurgica-Loma Dorada",t:180},
  {id:"S-Cantaros",c:"Siderurgica",n:"Siderurgica-Cantaros",t:180},
  {id:"U-Agaves",c:"Urrea",n:"Urrea-Agaves",t:185},
  {id:"U-Cima",c:"Urrea",n:"Urrea-Cima Serena",t:185},
  {id:"U-Salto",c:"Urrea",n:"Urrea-El salto",t:185},
  {id:"U-Oblatos",c:"Urrea",n:"Urrea-Oblatos",t:185},
  {id:"U-Ocotlan",c:"Urrea",n:"Urrea-Ocotlan",t:250},
  {id:"U-Polanco",c:"Urrea",n:"Urrea-Polanco",t:185},
  {id:"U-Sabinos",c:"Urrea",n:"Urrea-Sabinos",t:185},
  {id:"U-SantaCruz",c:"Urrea",n:"Urrea-Santa cruz del Valle",t:185},
  {id:"U-Tlajomulco",c:"Urrea",n:"Urrea-Tlajomulco",t:185},
  {id:"U-Tonala",c:"Urrea",n:"Urrea-Tonala",t:185},
  {id:"U-Villas",c:"Urrea",n:"Urrea-Villas andalucia",t:185},
  {id:"U-ADMTransito",c:"Urrea",n:"Urrea-ADM TRANSITO",t:185},
  {id:"U-ADMPeri",c:"Urrea",n:"Urrea-ADM PERIFERICO",t:185},
  {id:"U-ADMArt",c:"Urrea",n:"Urrea-ADM ARTESANOS",t:185},
];

const DAYS = ["JUE","VIE","SÁB","DOM","LUN","MAR","MIÉ"];
const DFULL = ["Jueves","Viernes","Sábado","Domingo","Lunes","Martes","Miércoles"];

// ══ HELPERS ══
function todayStr() {
  const d = new Date();
  const y = d.getFullYear();
  const m = String(d.getMonth()+1).padStart(2,'0');
  const day = String(d.getDate()).padStart(2,'0');
  return `${y}-${m}-${day}`;
}

async function sbFetch(table, qs = "") {
  const res = await fetch(`${SB_URL}/rest/v1/${table}${qs}`, {
    headers: {
      "apikey": SB_KEY,
      "Authorization": `Bearer ${SB_KEY}`,
    }
  });
  if (!res.ok) throw new Error(`Supabase error: ${await res.text()}`);
  return res.json();
}

async function getSemanaActiva() {
  const rows = await sbFetch('rv_semanas', '?activa=eq.true&order=week_start.desc&limit=1');
  return rows && rows.length ? rows[0] : null;
}

async function getViajesSemana(semanaId) {
  return await sbFetch('rv_viajes', `?semana_id=eq.${semanaId}&order=fecha,hora`);
}

async function getTarifas() {
  try {
    const rows = await sbFetch('rv_config', '?clave=eq.tarifas_choferes&limit=1');
    if (rows && rows.length) return rows[0].valor;
  } catch(e) {}
  const def = {};
  RUTAS.forEach(r => def[r.id] = r.t);
  return def;
}

function calcNominaEstimada(viajes, tarifas) {
  const map = {};
  viajes.filter(v => v.confirmacion === 'SI').forEach(v => {
    const r = RUTAS.find(x => x.id === v.ruta_id);
    if (!r) return;
    const tarifa = tarifas[r.id] || r.t;
    const medio = tarifa / 2;
    const ce = v.chofer_entrada || '', cs = v.chofer_salida || '';
    const comp = ce && cs && ce === cs;
    if (comp) {
      if (!map[ce]) map[ce] = 0;
      map[ce] += tarifa;
    } else {
      if (ce) { if (!map[ce]) map[ce] = 0; map[ce] += medio; }
      if (cs && cs !== ce) { if (!map[cs]) map[cs] = 0; map[cs] += medio; }
    }
  });
  return Object.values(map).reduce((s, v) => s + v, 0);
}

function fmtMoney(n) {
  return '$' + n.toLocaleString('es-MX', { minimumFractionDigits: 2 });
}

async function handleEstatusHoy() {
  const semana = await getSemanaActiva();
  if (!semana) return "⚠️ No hay semana activa registrada.";
  const viajes = await getViajesSemana(semana.id);
  const today = todayStr();
  const hoy = viajes.filter(v => v.fecha === today);
  const conf = hoy.filter(v => v.confirmacion === 'SI').length;
  const pend = hoy.length - conf;
  const pct = hoy.length ? Math.round(conf / hoy.length * 100) : 0;

  return `📊 *Estatus de HOY*\n\n` +
    `Total viajes: ${hoy.length}\n` +
    `✅ Confirmados: ${conf}\n` +
    `⏳ Pendientes: ${pend}\n` +
    `Avance: ${pct}%`;
}

async function handlePendientesHoy() {
  const semana = await getSemanaActiva();
  if (!semana) return "⚠️ No hay semana activa registrada.";
  const viajes = await getViajesSemana(semana.id);
  const today = todayStr();
  const pendientes = viajes.filter(v => v.fecha === today && v.confirmacion === 'NO');

  if (!pendientes.length) return "✅ No hay pendientes hoy. Todo confirmado.";

  let msg = `⏳ *Pendientes de HOY* (${pendientes.length})\n\n`;
  pendientes.slice(0, 15).forEach(v => {
    const r = RUTAS.find(x => x.id === v.ruta_id);
    msg += `• ${v.hora || '—'} ${r ? r.n : v.ruta_id} — ${v.chofer_entrada || 'sin asignar'}\n`;
  });
  if (pendientes.length > 15) msg += `\n...y ${pendientes.length - 15} más.`;
  return msg;
}

async function handleEstatusDia(diaAbbr) {
  const semana = await getSemanaActiva();
  if (!semana) return "⚠️ No hay semana activa registrada.";
  const viajes = await getViajesSemana(semana.id);
  const idx = DAYS.indexOf(diaAbbr);
  if (idx < 0) return "No entendí el día. Usa: jueves, viernes, sabado, domingo, lunes, martes, miercoles.";

  const delDia = viajes.filter(v => v.dia === diaAbbr);
  const conf = delDia.filter(v => v.confirmacion === 'SI').length;
  const pend = delDia.length - conf;

  return `📅 *${DFULL[idx]}*\n\n` +
    `Total viajes: ${delDia.length}\n` +
    `✅ Confirmados: ${conf}\n` +
    `⏳ Pendientes: ${pend}`;
}

async function handleResumenSemana() {
  const semana = await getSemanaActiva();
  if (!semana) return "⚠️ No hay semana activa registrada.";
  const viajes = await getViajesSemana(semana.id);

  let msg = `🗓️ *Resumen Semana* (${semana.week_start})\n\n`;
  DAYS.forEach((d) => {
    const delDia = viajes.filter(v => v.dia === d);
    const conf = delDia.filter(v => v.confirmacion === 'SI').length;
    msg += `${d}: ${conf}/${delDia.length} confirmados\n`;
  });
  const totalConf = viajes.filter(v => v.confirmacion === 'SI').length;
  msg += `\n*Total semana:* ${totalConf}/${viajes.length} confirmados`;
  return msg;
}

async function handleNominaEstimada() {
  const semana = await getSemanaActiva();
  if (!semana) return "⚠️ No hay semana activa registrada.";
  const viajes = await getViajesSemana(semana.id);
  const tarifas = await getTarifas();
  const total = calcNominaEstimada(viajes, tarifas);
  const confirmados = viajes.filter(v => v.confirmacion === 'SI').length;

  return `💰 *Nómina Estimada* (semana en curso)\n\n` +
    `Basada en ${confirmados} viajes confirmados.\n` +
    `Total estimado: ${fmtMoney(total)}\n\n` +
    `_Nota: el cálculo final puede variar si hay deducciones activas — revisa la app para el neto exacto._`;
}

function handleAyuda() {
  return `🤖 *Bot Ruiz Vitela — Comandos disponibles*\n\n` +
    `*estatus* — resumen de hoy\n` +
    `*pendientes* — viajes sin confirmar hoy\n` +
    `*semana* — resumen de toda la semana\n` +
    `*nomina* — estimado de nómina actual\n` +
    `*jueves* / *viernes* / *sabado* / etc — estatus de un día específico\n\n` +
    `Escribe cualquiera de estas palabras.`;
}

async function routeMessage(text) {
  const t = text.trim().toLowerCase();

  if (t.includes('ayuda') || t === 'help' || t === '?') return handleAyuda();
  if (t.includes('pendient')) return handlePendientesHoy();
  if (t.includes('nomina') || t.includes('nómina')) return handleNominaEstimada();
  if (t.includes('semana')) return handleResumenSemana();
  if (t.includes('estatus') || t.includes('status') || t.includes('hoy')) return handleEstatusHoy();

  const diaMap = {
    'jueves': 'JUE', 'viernes': 'VIE', 'sabado': 'SÁB', 'sábado': 'SÁB',
    'domingo': 'DOM', 'lunes': 'LUN', 'martes': 'MAR', 'miercoles': 'MIÉ', 'miércoles': 'MIÉ'
  };
  for (const [key, val] of Object.entries(diaMap)) {
    if (t.includes(key)) return handleEstatusDia(val);
  }

  return `No entendí ese comando. Escribe *ayuda* para ver las opciones disponibles.`;
}

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(200).send('Webhook activo. Envía POST desde Twilio.');
  }

  try {
    const from = req.body.From;
    const body = req.body.Body || '';

    if (ALLOWED_NUMBERS.length > 0 && !ALLOWED_NUMBERS.includes(from)) {
      const twiml = `<?xml version="1.0" encoding="UTF-8"?>
<Response><Message>🚫 No autorizado para usar este servicio.</Message></Response>`;
      res.setHeader('Content-Type', 'text/xml');
      return res.status(200).send(twiml);
    }

    const reply = await routeMessage(body);

    const escapedReply = reply
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;');

    const twiml = `<?xml version="1.0" encoding="UTF-8"?>
<Response><Message>${escapedReply}</Message></Response>`;

    res.setHeader('Content-Type', 'text/xml');
    return res.status(200).send(twiml);

  } catch (err) {
    console.error('Webhook error:', err);
    const twiml = `<?xml version="1.0" encoding="UTF-8"?>
<Response><Message>⚠️ Error al consultar datos. Intenta de nuevo en un momento.</Message></Response>`;
    res.setHeader('Content-Type', 'text/xml');
    return res.status(200).send(twiml);
  }
}
