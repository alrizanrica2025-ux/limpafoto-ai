# limpafoto-ai

<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>LimpaFoto IA</title>
<style>
*{box-sizing:border-box}
body{margin:0;font-family:Arial,sans-serif;background:#101323;color:#fff}
header{text-align:center;padding:22px 12px;background:#25204c}
header h1{margin:0;color:#a99bff}
header p{margin:8px 0 0;color:#ddd}
main{max-width:850px;margin:auto;padding:16px}
.panel{background:#1c2034;padding:16px;border-radius:16px;margin-bottom:15px}
#area{background:#111827;min-height:180px;display:flex;align-items:center;justify-content:center;border:1px dashed #8274ff;border-radius:12px;overflow:hidden}
canvas{display:none;max-width:100%;height:auto;touch-action:none;cursor:crosshair}
input[type=file]{display:none}
button,.upload{display:inline-block;border:0;border-radius:10px;padding:12px 14px;margin:4px;background:#5144c7;color:white;font-weight:bold;font-size:14px;cursor:pointer}
button:disabled{opacity:.4}
.green{background:#159b76}
.gray{background:#424960}
.tools{display:flex;flex-wrap:wrap;gap:5px;margin-top:12px}
label{display:block;margin:12px 0 6px}
input[type=range]{width:100%}
#status{padding:12px;background:#292543;border-radius:10px;margin-top:12px;color:#d8d1ff}
small{color:#c1c5d5;line-height:1.5}
</style>
</head>
<body>
<header>
<h1>✦ LimpaFoto IA</h1>
<p>Edite suas fotos diretamente no celular</p>
</header>

<main>
<section class="panel">
<h3>1. Escolha sua foto</h3>
<label class="upload" for="arquivo">📷 Selecionar foto</label>
<input id="arquivo" type="file" accept="image/*">

<div id="area">
<div id="mensagem">Sua foto aparecerá aqui</div>
<canvas id="foto"></canvas>
</div>

<div class="tools">
<button id="marcar" disabled>✎ Marcar área</button>
<button id="limpar" class="green" disabled>Reconstruir área</button>
<button id="desfazer" class="gray" disabled>↶ Desfazer</button>
<button id="salvar" disabled>⬇ Salvar foto</button>
</div>

<label for="pincel">Tamanho do pincel: <span id="valor">25</span> px</label>
<input id="pincel" type="range" min="5" max="100" value="25">

<div id="status" role="status">Selecione uma foto para começar.</div>
</section>

<section class="panel">
<h3>Como usar</h3>
<p>1. Selecione uma foto sua ou que você tenha autorização para editar.</p>
<p>2. Toque em “Marcar área”.</p>
<p>3. Passe o dedo sobre o texto ou a marca que deseja remover.</p>
<p>4. Toque em “Reconstruir área”.</p>
<p>5. Confira o resultado e toque em “Salvar foto”.</p>
<small>Esta versão funciona sem chave de API. A reconstrução usa pixels vizinhos, não IA generativa, e pode deixar marcas em fundos complexos.</small>
</section>
</main>

<script>
const $=id=>document.getElementById(id);
const canvas=$("foto");
const ctx=canvas.getContext("2d",{willReadFrequently:true});
let original=null,imagem=null,marcacao=null;
let desenhando=false,modo=false,anterior=null;

function status(texto){$("status").textContent=texto}
function habilitar(){
 ["marcar","limpar","desfazer","salvar"].forEach(id=>{
  $(id).disabled=!imagem;
 });
}
$("arquivo").addEventListener("change",e=>{
 const arquivo=e.target.files[0];
 if(!arquivo)return;
 if(!arquivo.type.startsWith("image/")){
  status("Selecione um arquivo de imagem válido.");return;
 }
 const url=URL.createObjectURL(arquivo);
 const img=new Image();
 img.onload=()=>{
  const escala=Math.min(1,1800/Math.max(img.naturalWidth,img.naturalHeight));
  canvas.width=Math.round(img.naturalWidth*escala);
  canvas.height=Math.round(img.naturalHeight*escala);
  ctx.drawImage(img,0,0,canvas.width,canvas.height);
  original=ctx.getImageData(0,0,canvas.width,canvas.height);
  imagem=true;
  marcacao=new Uint8Array(canvas.width*canvas.height);
  anterior=null;
  canvas.style.display="block";
  $("mensagem").style.display="none";
  habilitar();
  $("desfazer").disabled=true;
  status("Foto carregada. Marque a área desejada.");
  URL.revokeObjectURL(url);
 };
 img.onerror=()=>status("Não foi possível abrir essa imagem.");
 img.src=url;
});
$("pincel").addEventListener("input",()=>{
 $("valor").textContent=$("pincel").value;
});
$("marcar").onclick=()=>{
 modo=!modo;
 $("marcar").textContent=modo?"Concluir marcação":"✎ Marcar área";
 canvas.style.cursor=modo?"crosshair":"default";
 status(modo?"Passe o dedo sobre a área a editar.":"Marcação concluída.");
};
function ponto(e){
 const r=canvas.getBoundingClientRect();
 return {
  x:(e.clientX-r.left)*canvas.width/r.width,
  y:(e.clientY-r.top)*canvas.height/r.height
 };
}
function pintar(x,y){
 const raio=Number($("pincel").value)/2;
 const w=canvas.width,h=canvas.height;
 for(let yy=Math.max(0,Math.floor(y-raio));yy<Math.min(h,y+raio);yy++){
  for(let xx=Math.max(0,Math.floor(x-raio));xx<Math.min(w,x+raio);xx++){
   if((xx-x)**2+(yy-y)**2<=raio**2){
    marcacao[yy*w+xx]=1;
   }
  }
 }
 mostrarMarcacao();
}
function mostrarMarcacao(){
 if(!original)return;
 const dados=new ImageData(
  new Uint8ClampedArray(original.data),
  canvas.width,canvas.height
 );
 for(let i=0;i<marcacao.length;i++){
  if(marcacao[i]){
   const p=i*4;
   dados.data[p]=160;
   dados.data[p+1]=70;
   dados.data[p+2]=255;
   dados.data[p+3]=255;
  }
 }
 ctx.putImageData(dados,0,0);
}
canvas.addEventListener("pointerdown",e=>{
 if(!modo||!imagem)return;
 desenhando=true;
 canvas.setPointerCapture(e.pointerId);
 const p=ponto(e);pintar(p.x,p.y);
});
canvas.addEventListener("pointermove",e=>{
 if(!desenhando)return;
 const p=ponto(e);pintar(p.x,p.y);
});
function terminar(){desenhando=false}
canvas.addEventListener("pointerup",terminar);
canvas.addEventListener("pointercancel",terminar);

$("limpar").onclick=()=>{
 if(!imagem)return;
 const w=canvas.width,h=canvas.height;
 let total=0;
 for(const v of marcacao)total+=v;
 if(!total){
  status("Primeiro marque a área que deseja editar.");return;
 }
 anterior=ctx.getImageData(0,0,w,h);
 const base=original.data;
 const saida=new Uint8ClampedArray(base);
 for(let y=0;y<h;y++){
  for(let x=0;x<w;x++){
   const i=y*w+x;
   if(!marcacao[i])continue;
   let soma=[0,0,0,0],n=0;
   for(let d=3;d<=30;d+=3){
    const vizinhos=[
     [x-d,y],[x+d,y],[x,y-d],[x,y+d]
    ];
    for(const [xx,yy] of vizinhos){
     if(xx<0||xx>=w||yy<0||yy>=h)continue;
     if(marcacao[yy*w+xx])continue;
     const q=(yy*w+xx)*4;
     for(let c=0;c<4;c++)soma[c]+=base[q+c];
     n++;
    }
    if(n>=4)break;
   }
   if(n){
    for(let c=0;c<4;c++)saida[i*4+c]=soma[c]/n;
   }
  }
 }
 original=new ImageData(saida,w,h);
 ctx.putImageData(original,0,0);
 marcacao.fill(0);
 modo=false;
 $("marcar").textContent="✎ Marcar área";
 $("desfazer").disabled=false;
 status("Área reconstruída com pixels vizinhos. Confira a foto.");
};
$("desfazer").onclick=()=>{
 if(!anterior)return;
 ctx.putImageData(anterior,0,0);
 original=ctx.getImageData(0,0,canvas.width,canvas.height);
 marcacao.fill(0);
 anterior=null;
 $("desfazer").disabled=true;
 status("Última edição desfeita.");
};
$("salvar").onclick=()=>{
 if(!imagem)return;
 const link=document.createElement("a");
 link.download="limpafoto-editada.png";
 link.href=canvas.toDataURL("image/png");
 link.click();
};
</script>
</body>
</html>
