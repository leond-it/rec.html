[rec.html](https://github.com/user-attachments/files/33317549/rec.html)
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>讯飞语音听写测试</title>
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{padding:30px;font-family:system-ui}
#result{width:100%;height:200px;margin:20px 0;padding:10px;border:1px solid #ccc}
button{padding:10px 20px;margin-right:10px;cursor:pointer}
</style>
</head>
<body>
<h3>讯飞实时语音转文字</h3>
<div>
<button id="startBtn">开始录音</button>
<button id="stopBtn" disabled>停止录音</button>
</div>
<textarea id="result" placeholder="识别文字会出现在这里..." readonly></textarea>

<script>
// ========== 你的讯飞信息 ==========
const APPID = "78c28e82";
const APIKey = "d9bbaad458d9fff05a28918c2da5c28f";
const APISecret = "N2I2M2NkNTc2NTdmM2FjZDMyODA5Mzc";
const wsUrl = "wss://iat-api.xfyun.cn/v2/iat";

let ws;
let recorder;
let audioContext;
let resultText = "";
const resultDom = document.getElementById("result");
const startBtn = document.getElementById("startBtn");
const stopBtn = document.getElementById("stopBtn");

// 生成鉴权url
async function getAuthUrl(){
  const date = new Date().toUTCString();
  const signatureOrigin = `host: iat-api.xfyun.cn\ndate: ${date}\nGET /v2/iat HTTP/1.1`;
  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey("raw", encoder.encode(APISecret), {name:"HMAC",hash:"SHA-256"},false,["sign"]);
  const signatureBuf = await crypto.subtle.sign("HMAC",key,encoder.encode(signatureOrigin));
  const signature = btoa(String.fromCharCode(...new Uint8Array(signatureBuf)));
  const authorization = btoa(`api_key="${APIKey}",algorithm="hmac-sha256",headers="host date request-line",signature="${signature}"`);
  const params = new URLSearchParams({authorization,date,host:"iat-api.xfyun.cn"});
  return `${wsUrl}?${params.toString()}`;
}

// 音频处理：拿到16k单声道PCM
function startRecord(){
  return navigator.mediaDevices.getUserMedia({audio:true})
  .then(stream=>{
    audioContext = new AudioContext({sampleRate:16000});
    const source = audioContext.createMediaStreamSource(stream);
    const processor = audioContext.createScriptProcessor(4096,1,1);
    source.connect(processor);
    processor.connect(audioContext.destination);
    processor.onaudioprocess = e=>{
      if(!ws || ws.readyState!==WebSocket.OPEN) return;
      const input = e.inputBuffer.getChannelData(0);
      const out = new Int16Array(input.length);
      for(let i=0;i<input.length;i++){
        const s = Math.max(-1,Math.min(1,input[i]));
        out[i] = s < 0 ? s * 0x8000 : s * 0x7FFF;
      }
      ws.send(JSON.stringify({
        common:{app_id:APPID},
        business:{language:"zh_cn",domain:"iat",vad_eos:1000},
        data:{status:1,audio:btoa(String.fromCharCode(...new Uint8Array(out.buffer)))}
      }))
    }
    recorder = {stream,processor,source};
  })
}

startBtn.onclick = async ()=>{
  resultText = "";
  resultDom.value = "";
  startBtn.disabled = true;
  stopBtn.disabled = false;
  const url = await getAuthUrl();
  ws = new WebSocket(url);
  ws.onopen = ()=>{
    ws.send(JSON.stringify({
      common:{app_id:APPID},
      business:{language:"zh_cn",domain:"iat",vad_eos:1000},
      data:{status:0}
    }))
  }
  ws.onmessage = (evt)=>{
    const res = JSON.parse(evt.data);
    if(res.code!==0) return;
    const words = res.data.result?.ws || [];
    let str = "";
    words.forEach(w=>w.cw.forEach(c=>str+=c.w));
    resultText += str;
    resultDom.value = resultText;
  }
  ws.onerror = e=>alert("连接失败");
  ws.onclose = ()=>{}
  await startRecord();
}

stopBtn.onclick = ()=>{
  if(ws) ws.send(JSON.stringify({data:{status:2}}));
  if(recorder?.processor) recorder.processor.disconnect();
  if(recorder?.stream) recorder.stream.getTracks().forEach(t=>t.stop());
  if(audioContext) audioContext.close();
  startBtn.disabled = false;
  stopBtn.disabled = true;
}
</script>
</body>
</html>
