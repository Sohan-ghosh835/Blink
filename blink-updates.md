# blink: updates from bokeh to the latest

Live app: https://claude.ai/artifact/Wd3LMF4RXP6gFrBGhum3hH
The complete, final `blink.html` is delivered as a separate file. It already contains everything below.
The snippets are the exact final code for each update, so you can see what each one added.

## 1. Bokeh, pass 1: light-source detection (replaced later)

First rewrite: found the brightest spots in the frame and put soft discs on them. It flickered and treated bright walls as lights, so it was replaced by pass 2. Its code no longer exists in the file.


## 2. Bokeh, pass 2: real-lens recipe + no flicker (extended later)

Switched to how a real lens works: a bright-pass mask (only small pixels much brighter than their surroundings), each light spread into a disc that keeps its own colour with a pink rim, plus blending with the previous frame so the live preview doesn't jitter. The helpers below are still used by the final version.

**JS: helpers (canvas pool, box blur, live flag), still in the file**

````js
let LIVE=false,zoom=1;const BK={};
['S','H','Hp','N','P','O','M','T'].forEach(k=>{BK[k]=document.createElement('canvas');BK[k+'x']=BK[k].getContext('2d',{willReadFrequently:k==='S'||k==='T'})});
const fit=(k,W,H)=>{if(BK[k].width!==W||BK[k].height!==H){BK[k].width=W;BK[k].height=H}};
function box(L,w,h,r){const T=new Float32Array(w*h),O=new Float32Array(w*h),k=2*r+1,cl=(v,m)=>Math.min(m,Math.max(0,v));
 for(let y=0;y<h;y++){let s=0;const o=y*w;for(let x=-r;x<=r;x++)s+=L[o+cl(x,w-1)];
  for(let x=0;x<w;x++){T[o+x]=s/k;s+=L[o+cl(x+r+1,w-1)]-L[o+cl(x-r,w-1)]}}
 for(let x=0;x<w;x++){let s=0;for(let y=-r;y<=r;y++)s+=T[cl(y,h-1)*w+x];
  for(let y=0;y<h;y++){O[y*w+x]=s/k;s+=T[cl(y+r+1,h-1)*w+x]-T[cl(y-r,h-1)*w+x]}}
 return O}
````

**JS: the live loop marks the preview as LIVE so bokeh can smooth over time**

````js
function loop(){
 if(vid.videoWidth&&!$('#cam').hidden){cover(pc,vid,vid.videoWidth,vid.videoHeight,480,640,face==='user',zoom);LIVE=true;fx(pc,480,640,stack);LIVE=false}
 requestAnimationFrame(loop)}
````


## 3. Bokeh, pass 3 (current): person mask, background-only bokeh, stricter lights

Estimates where the person is (centre colour model vs border colour model + a centre prior), blurs only the background, ignores anything inside the person mask (forehead shine, bright clothes), uses a stricter light test, and draws the sharp person on top of the discs.

**JS: person-mask estimate (paste after the box() helper)**

````js
/* subject (person) estimate: colour models of a centre seed vs border seed + a centre-weighted prior, no ML download needed */
function subjMask(d,w,h,prev){
 const N=w*h,bin=new Uint16Array(N),F=new Float32Array(512),G=new Float32Array(512),S=new Float32Array(N);let nf=0,ng=0;
 for(let y=0;y<h;y++)for(let x=0;x<w;x++){const i=y*w+x,o=i*4,u=x/w,v=y/h,b=(d[o]>>5)<<6|(d[o+1]>>5)<<3|(d[o+2]>>5);bin[i]=b;
  if(((u-.5)/.16)**2+((v-.48)/.22)**2<1||(v>.8&&Math.abs(u-.5)<.26)){F[b]++;nf++}
  else if(((u<.1||u>.9)&&v<.75)||v<.1){G[b]++;ng++}}
 for(let y=0;y<h;y++)for(let x=0;x<w;x++){const i=y*w+x,u=x/w,v=y/h,pf=F[bin[i]]/nf+.002,pg=G[bin[i]]/ng+.002;
  S[i]=.7*pf/(pf+pg)+.3*Math.exp(-(((u-.5)/.3)**2+((v-.62)/.45)**2)/2)}
 const M=box(box(S,w,h,2),w,h,2),m=new Float32Array(N);let t=0;
 for(let i=0;i<N;i++){const q=Math.min(1,Math.max(0,(M[i]-.47)/.2));m[i]=q*q*(3-2*q);t+=m[i]}
 if(t/N<.06||t/N>.92)for(let y=0;y<h;y++)for(let x=0;x<w;x++){const u=x/w,v=y/h;m[y*w+x]=Math.min(1,Math.max(0,1.6-Math.hypot((u-.5)/.28,(v-.55)/.42)))}
 if(prev&&prev.length===N)for(let i=0;i<N;i++)m[i]=.6*prev[i]+.4*m[i];
 return m}
````

**JS: the bokeh filter entry inside the FX object**

````js
 bokeh:(c,w,h)=>{
  /* 1) find the person  2) blur only the background  3) turn real light sources (small, much brighter than surroundings) into discs  4) put the sharp person back on top */
  const sw=w>>1,sh=h>>1,mw=80,mh=Math.round(80*h/w),sm=t=>t<=0?0:t>=1?1:t*t*(3-2*t),bl=blur(c,w,h,8);
  for(const k of ['S','H','Hp','N'])fit(k,sw,sh);fit('O',w,h);fit('M',mw,mh);fit('T',mw,mh);
  const fresh=BK.P.width!==sw||BK.P.height!==sh;if(fresh){BK.P.width=sw;BK.P.height=sh}
  const {Sx,Hx,Hpx,Nx,Px,Ox,Mx,Tx}=BK;Sx.drawImage(c.canvas,0,0,sw,sh);
  Tx.drawImage(BK.S,0,0,mw,mh);const mk=BK.mk=subjMask(Tx.getImageData(0,0,mw,mh).data,mw,mh,LIVE?BK.mk:null);
  const mi=Mx.createImageData(mw,mh);for(let i=0;i<mk.length;i++){mi.data[i*4]=mi.data[i*4+1]=mi.data[i*4+2]=255;mi.data[i*4+3]=mk[i]*255}Mx.putImageData(mi,0,0);
  Ox.globalCompositeOperation='source-over';Ox.clearRect(0,0,w,h);Ox.drawImage(c.canvas,0,0);Ox.globalCompositeOperation='destination-in';Ox.imageSmoothingQuality='high';Ox.drawImage(BK.M,0,0,w,h);
  const im=Sx.getImageData(0,0,sw,sh),d=im.data,L=new Float32Array(sw*sh),M=box((()=>{for(let i=0,j=0;j<L.length;i+=4,j++)L[j]=lum(d,i);return L})(),sw,sh,Math.round(sw/24));
  for(let i=0,j=0;j<L.length;i+=4,j++){const l=L[j],x=j%sw,y=(j/sw)|0,inS=mk[Math.min(mh-1,y*mh/sh|0)*mw+Math.min(mw-1,x*mw/sw|0)],
   s=sm((l-M[j]-45)/60)*sm((l-190)/45)*1.2*(1-inS);
   d[i]=mix(l,d[i],1.5)*s;d[i+1]=mix(l,d[i+1],1.5)*s;d[i+2]=mix(l,d[i+2],1.5)*s;d[i+3]=255}
  Hx.putImageData(im,0,0);
  Hpx.globalCompositeOperation='source-over';Hpx.drawImage(BK.H,0,0);Hpx.globalCompositeOperation='multiply';Hpx.fillStyle='rgb(255,130,215)';Hpx.fillRect(0,0,sw,sh);
  Nx.globalCompositeOperation='source-over';Nx.fillStyle='#000';Nx.fillRect(0,0,sw,sh);Nx.globalCompositeOperation='lighter';
  const R=sw*.055;
  [[R,24,BK.Hp,.75],[R*.66,14,BK.H,.5],[R*.33,7,BK.H,.5],[0,1,BK.H,.5]].forEach(([r,n,src,a])=>{Nx.globalAlpha=a;
   for(let i=0;i<n;i++){const t=6.283*i/n;Nx.drawImage(src,Math.cos(t)*r,Math.sin(t)*r)}});
  Nx.globalAlpha=1;
  if(LIVE&&!fresh){Px.globalAlpha=.4;Px.drawImage(BK.N,0,0);Px.globalAlpha=1}else Px.drawImage(BK.N,0,0);
  c.save();c.globalAlpha=.92;c.drawImage(bl,0,0,w,h);c.globalAlpha=1;c.fillStyle='rgba(0,0,0,.14)';c.fillRect(0,0,w,h);
  c.globalCompositeOperation='screen';c.imageSmoothingQuality='high';c.drawImage(BK.P,0,0,w,h);c.restore();
  c.drawImage(BK.O,0,0)},
````


## 4. Live zoom in the camera

Zoom bar on the preview (1x to 4x), pinch on touch screens, mouse wheel on desktop. Zoom is applied when cropping the camera feed, so the preview, saved photos and strip shots all match.

**CSS**

````css
.stage{touch-action:none}
.zoom{position:absolute;left:14px;right:14px;bottom:12px;display:flex;gap:10px;align-items:center;background:rgba(0,0,0,.6);border:1px solid var(--ln);border-radius:99px;padding:5px 14px;font-size:12px;font-weight:700;color:var(--pk2)}
.zoom span{min-width:34px}
````

**HTML: inside the .stage div, after the flash div**

````html
<div class="zoom"><span id="zl">1.0×</span><input type="range" id="zm" min="1" max="4" step=".05" value="1" aria-label="Zoom"></div>
````

**JS: cover() now takes a zoom value**

````js
function cover(c,src,sw,sh,w,h,mir,z=1){const ar=w/h;let cw=sw,ch=sh;sw/sh>ar?cw=sh*ar:ch=sw/ar;cw/=z;ch/=z;
 c.save();if(mir){c.translate(w,0);c.scale(-1,1)}c.drawImage(src,(sw-cw)/2,(sh-ch)/2,cw,ch,0,0,w,h);c.restore()}
````

**JS: zoom controls (slider, pinch, wheel)**

````js
function setZoom(z){zoom=Math.min(4,Math.max(1,z));$('#zm').value=zoom;$('#zl').textContent=zoom.toFixed(1)+'×'}
$('#zm').oninput=e=>setZoom(+e.target.value);
{const st=document.querySelector('.stage');let pd=0;
 st.addEventListener('wheel',e=>{e.preventDefault();setZoom(zoom*(e.deltaY<0?1.08:.93))},{passive:false});
 st.addEventListener('touchmove',e=>{if(e.touches.length===2){const t=e.touches,d=Math.hypot(t[0].clientX-t[1].clientX,t[0].clientY-t[1].clientY);if(pd)setZoom(zoom*d/pd);pd=d;e.preventDefault()}},{passive:false});
 st.addEventListener('touchend',()=>pd=0)}
````

**JS: snap() passes the zoom**

````js
function snap(){const vw=vid.videoWidth,vh=vid.videoHeight,w=Math.min(1080,Math.floor(Math.min(vw,vh*.75))),h=Math.round(w*4/3),c=document.createElement('canvas');
 c.width=w;c.height=h;cover(c.getContext('2d'),vid,vw,vh,w,h,face==='user',zoom);return c}
````

**JS: loop() passes the zoom**

````js
function loop(){
 if(vid.videoWidth&&!$('#cam').hidden){cover(pc,vid,vid.videoWidth,vid.videoHeight,480,640,face==='user',zoom);LIVE=true;fx(pc,480,640,stack);LIVE=false}
 requestAnimationFrame(loop)}
````


## 5. Zoom and crop in the editor (preset shapes, drag to move)

Crop shapes 3:4, 1:1, 4:5, 9:16 and 16:9, a zoom slider, drag the photo to reposition, mouse wheel to zoom. The crop is applied before filters, so effects fit the new shape.

**CSS**

````css
.ars{display:flex;gap:6px;align-items:center;flex:none;font-size:11px;letter-spacing:.14em;text-transform:uppercase;color:#a1738c}
.ars span{margin-right:6px}
.ar{border-radius:8px;padding:5px 10px;font-size:12px}
.ar.sel{border-color:var(--pk);color:var(--pk)}
#ec{touch-action:none;cursor:grab}
````

**HTML: crop row (after the filter chips) and zoom slider (inside the .sl block)**

````html
<div class="ars" id="ars"><span>crop</span></div>

<label for="zc">zoom</label><input type="range" id="zc" min="1" max="4" step=".05" value="1">
````

**JS: crop model, cropRect() and paint()**

````js
const DC=()=>({r:.75,z:1,cx:.5,cy:.5,free:null});
function cropRect(p){const rw=p.raw.width,rh=p.raw.height,{r,z}=p.crop;if(p.crop.free){const f=p.crop.free;return[f.x*rw,f.y*rh,f.w*rw,f.h*rh]}let cw=rw,ch=rw/r;if(ch>rh){ch=rh;cw=rh*r}cw/=z;ch/=z;
 const x=Math.min(rw-cw,Math.max(0,p.crop.cx*rw-cw/2)),y=Math.min(rh-ch,Math.max(0,p.crop.cy*rh-ch/2));p.crop.cx=(x+cw/2)/rw;p.crop.cy=(y+ch/2)/rh;return[x,y,cw,ch]}
function paint(p,c){const [sx,sy,cw,ch]=cropRect(p),f=Math.max(1,720/Math.min(cw,ch));c.width=Math.round(cw*f);c.height=Math.round(ch*f);
 const x=c.getContext('2d',{willReadFrequently:true});x.imageSmoothingQuality='high';x.drawImage(p.raw,sx,sy,cw,ch,0,0,c.width,c.height);fx(x,c.width,c.height,p.stack,p.adj)}
````

**JS: new photos start with a default crop**

````js
const newPhoto=()=>({raw:snap(),stack:[...stack],adj:{b:0,c:1},crop:DC()});
````

**JS: editor redraw, thumbnail and edit()**

````js
function drawEd(){if(q)return;q=requestAnimationFrame(()=>{q=0;if(!cur||!photos.includes(cur))return;paintView();if($('#ed').classList.contains('crop'))return;
 const t=cur.th||(cur.th=document.createElement('canvas'));t.width=120;t.height=160;cover(t.getContext('2d'),ec,ec.width,ec.height,120,160,false);upThumb()})}
function edit(p){cur=p;$('#ed').classList.remove('crop');show('ed');$('#echips').mark();$('#br').value=p.adj.b;$('#ct').value=p.adj.c;syncCrop();drawEd()}
````

**JS: crop buttons, zoom slider, drag and wheel**

````js
const RATIOS=[['3:4',.75],['1:1',1],['4:5',.8],['9:16',9/16],['16:9',16/9]];
RATIOS.forEach(([l,r])=>{const b=document.createElement('button');b.className='b ar';b.textContent=l;b.dataset.r=r;b.onclick=()=>{cur.crop.free=null;cur.crop.r=r;syncCrop();drawEd()};$('#ars').append(b)});
function syncCrop(){$('#zc').value=cur.crop.z;$('#zc').disabled=!!cur.crop.free;document.querySelectorAll('.ar').forEach(b=>b.classList.toggle('sel',!cur.crop.free&&Math.abs(+b.dataset.r-cur.crop.r)<.001))}
$('#zc').oninput=e=>{cur.crop.z=+e.target.value;drawEd()};
{let dg=null;
 ec.onpointerdown=e=>{if(cur.crop.free)return;dg={x:e.clientX,y:e.clientY};ec.setPointerCapture(e.pointerId);ec.style.cursor='grabbing'};
 ec.onpointermove=e=>{if(!dg||!cur)return;const rc=ec.getBoundingClientRect(),[,,cw,ch]=cropRect(cur);
  cur.crop.cx-=(e.clientX-dg.x)/rc.width*cw/cur.raw.width;cur.crop.cy-=(e.clientY-dg.y)/rc.height*ch/cur.raw.height;dg={x:e.clientX,y:e.clientY};drawEd()};
 ec.onpointerup=ec.onpointercancel=()=>{dg=null;ec.style.cursor='grab'};
 ec.addEventListener('wheel',e=>{e.preventDefault();if(cur.crop.free)return;cur.crop.z=Math.min(4,Math.max(1,cur.crop.z*(e.deltaY<0?1.08:.93)));syncCrop();drawEd()},{passive:false})}
````


## 6. White photo strip

Third strip colour next to black and pink: white background with pink frames and a pink footer.

**HTML: button in the strip window**

````html
<button class="b" id="tp">pink strip</button><button class="b" id="tw">white strip</button>
````

**JS: strip colours, drawStrip() and button handlers**

````js
const TH={black:['#000','#ff3d9a'],pink:['#ff3d9a','#000'],white:['#fff','#ff3d9a']};
function drawStrip(){
 const W=600,P=44,F=W-2*P,G=24,H=P+4*F+3*G+170,c=$('#sc'),x=c.getContext('2d'),bg=TH[theme][0],ac=TH[theme][1];
 c.width=W;c.height=H;x.fillStyle=bg;x.fillRect(0,0,W,H);
 frames.forEach((f,i)=>{const y=P+i*(F+G);x.drawImage(f,0,(f.height-f.width)*.4,f.width,f.width,P,y,F,F);x.strokeStyle=ac;x.lineWidth=5;x.strokeRect(P,y,F,F)});
 x.fillStyle=ac;x.textAlign='center';x.font='800 76px Syne,system-ui,sans-serif';x.fillText('blink',W/2,H-90);
 x.font='600 22px Syne,system-ui,sans-serif';x.fillText(new Date().toLocaleDateString(undefined,{day:'numeric',month:'short',year:'numeric'}).toLowerCase(),W/2,H-46);
 for(const k of ['black','pink','white'])$('#t'+k[0]).classList.toggle('on',theme===k)}
$('#tb').onclick=()=>{theme='black';drawStrip()};
$('#tp').onclick=()=>{theme='pink';drawStrip()};
$('#tw').onclick=()=>{theme='white';drawStrip()};
````


## 7. Manual crop (drag a window over the photo)

Free-form crop window with 8 resize handles and a move-drag, thirds grid, dimmed outside area, apply / full photo / cancel. It starts from the current crop, and cropRect() (see section 5) returns the manual rectangle when one is set.

**CSS**

````css
.box{position:relative}
#co,#cbar,#chint{display:none}
#ed.crop #co{display:block}#ed.crop #cbar{display:flex}#ed.crop #chint{display:block}
#ed.crop .chips,#ed.crop .tabs,#ed.crop .ars,#ed.crop .sl,#ed.crop #mrow{display:none}
#co{position:absolute;overflow:hidden;touch-action:none}
#cr{position:absolute;border:2px solid var(--pk);box-shadow:0 0 0 9999px rgba(0,0,0,.62);cursor:move;
 background:linear-gradient(rgba(255,184,220,.35),rgba(255,184,220,.35)) 33.3% 0/1px 100% no-repeat,linear-gradient(rgba(255,184,220,.35),rgba(255,184,220,.35)) 66.6% 0/1px 100% no-repeat,linear-gradient(rgba(255,184,220,.35),rgba(255,184,220,.35)) 0 33.3%/100% 1px no-repeat,linear-gradient(rgba(255,184,220,.35),rgba(255,184,220,.35)) 0 66.6%/100% 1px no-repeat}
.hd{position:absolute;width:30px;height:30px;display:grid;place-items:center}
.hd::after{content:"";width:12px;height:12px;background:var(--pk);border:2px solid #000;border-radius:3px}
#chint{font-size:12px;color:var(--pk2);text-align:center;flex:none}
````

**HTML: editor photo box, hint, crop bar and buttons**

````html
<div class="box" id="eb"><canvas id="ec"></canvas><div id="co"><div id="cr"></div></div></div>
  <p id="chint">drag the window to move it, drag the handles to resize</p>
  
...
<div class="row" id="cbar"><button class="b pk" id="capply">apply crop</button><button class="b" id="cfull">full photo</button><button class="b" id="ccancel">cancel</button></div>
  <div class="row" id="mrow"><button class="b" id="back">camera</button><button class="b" id="undo">undo filter</button><button class="b" id="clr">reset</button><button class="b" id="del">delete photo</button><button class="b" id="mc">manual crop</button><button class="b pk" id="dl">download</button>
````

**JS: crop window logic**

````js
function paintView(){paint($('#ed').classList.contains('crop')?{raw:cur.raw,stack:cur.stack,adj:cur.adj,crop:DC()}:cur,ec)}
function dragRect(o,k,dx,dy){const m=.08;let {x,y,w,h}=o;
 if(k==='m'){x=Math.min(1-w,Math.max(0,x+dx));y=Math.min(1-h,Math.max(0,y+dy))}
 else{if(k.includes('w')){const nx=Math.min(o.x+o.w-m,Math.max(0,o.x+dx));w=o.x+o.w-nx;x=nx}
  if(k.includes('e'))w=Math.min(1-o.x,Math.max(m,o.w+dx));
  if(k.includes('n')){const ny=Math.min(o.y+o.h-m,Math.max(0,o.y+dy));h=o.y+o.h-ny;y=ny}
  if(k.includes('s'))h=Math.min(1-o.y,Math.max(m,o.h+dy))}
 return{x,y,w,h}}
{const co=$('#co'),cr=$('#cr');let cm=null,dr=null;
 [['nw',0,0],['n',50,0],['ne',100,0],['w',0,50],['e',100,50],['sw',0,100],['s',50,100],['se',100,100]].forEach(([k,X,Y])=>{const h=document.createElement('div');
  h.className='hd';h.dataset.h=k;h.style.left=X+'%';h.style.top=Y+'%';h.style.transform=`translate(-${X}%,-${Y}%)`;h.style.cursor=(k==='n'||k==='s')?'ns-resize':(k==='e'||k==='w')?'ew-resize':(k==='nw'||k==='se')?'nwse-resize':'nesw-resize';cr.append(h)});
 const draw=()=>{cr.style.left=cm.x*100+'%';cr.style.top=cm.y*100+'%';cr.style.width=cm.w*100+'%';cr.style.height=cm.h*100+'%'};
 const place=()=>{const r=ec.getBoundingClientRect(),b=$('#eb').getBoundingClientRect();co.style.left=r.left-b.left+'px';co.style.top=r.top-b.top+'px';co.style.width=r.width+'px';co.style.height=r.height+'px'};
 const leave=()=>{$('#ed').classList.remove('crop');syncCrop();drawEd()};
 $('#mc').onclick=()=>{const [x,y,w,h]=cropRect(cur),rw=cur.raw.width,rh=cur.raw.height;cm={x:x/rw,y:y/rh,w:w/rw,h:h/rh};
  $('#ed').classList.add('crop');paintView();place();draw()};
 $('#capply').onclick=()=>{cur.crop.free={...cm};cur.crop.z=1;leave()};
 $('#ccancel').onclick=leave;
 $('#cfull').onclick=()=>{cm={x:0,y:0,w:1,h:1};draw()};
 cr.onpointerdown=e=>{e.preventDefault();dr={k:e.target.dataset.h||'m',x:e.clientX,y:e.clientY,o:{...cm}};cr.setPointerCapture(e.pointerId)};
 cr.onpointermove=e=>{if(!dr)return;const b=co.getBoundingClientRect();cm=dragRect(dr.o,dr.k,(e.clientX-dr.x)/b.width,(e.clientY-dr.y)/b.height);draw()};
 cr.onpointerup=cr.onpointercancel=()=>dr=null;
 addEventListener('resize',()=>{if($('#ed').classList.contains('crop')){place();draw()}})}
````

