<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>간단한 사진게시판</title>
  <style>
    :root{--bg:#f7f7f8;--card:#ffffff;--accent:#2b6cb0}
    body{font-family:system-ui,-apple-system,Segoe UI,Roboto,'Noto Sans KR',sans-serif;margin:0;background:var(--bg);color:#111}
    header{background:var(--card);padding:16px 20px;display:flex;align-items:center;justify-content:space-between;box-shadow:0 1px 4px rgba(0,0,0,.06)}
    h1{font-size:18px;margin:0}
    main{padding:18px;max-width:1000px;margin:20px auto}
    .controls{display:flex;gap:8px;align-items:center}
    button{background:var(--accent);color:#fff;border:0;padding:8px 12px;border-radius:8px;cursor:pointer}
    .gallery{display:grid;grid-template-columns:repeat(auto-fill,minmax(180px,1fr));gap:12px}
    .card{background:var(--card);border-radius:10px;overflow:hidden;box-shadow:0 4px 12px rgba(0,0,0,.06);display:flex;flex-direction:column}
    .card img{width:100%;height:180px;object-fit:cover;display:block}
    .card .meta{padding:8px 10px;display:flex;justify-content:space-between;align-items:center;font-size:13px}
    .card .meta .time{color:#666}
    .card .actions{display:flex;gap:6px}
    .small{background:#eee;color:#333;padding:6px;border-radius:6px;font-size:12px;border:0}

    /* 모달 스타일 */
    .modal-backdrop{position:fixed;inset:0;background:rgba(0,0,0,.45);display:flex;align-items:center;justify-content:center;z-index:30}
    .modal{width:420px;max-width:94%;background:var(--card);border-radius:10px;padding:16px;box-shadow:0 8px 30px rgba(0,0,0,.3)}
    .form-row{display:flex;flex-direction:column;gap:6px;margin-bottom:10px}
    label{font-size:13px;color:#333}
    input[type="text"],input[type="file"]{padding:8px;border-radius:8px;border:1px solid #ddd;font-size:14px}
    .btn-row{display:flex;justify-content:flex-end;gap:8px;margin-top:8px}

    /* 전체보기 모달 */
    .viewer img{max-width:90vw;max-height:80vh;border-radius:8px;display:block}
    .hidden{display:none}

    footer{max-width:1000px;margin:0 auto;padding:20px;text-align:center;color:#666}
  </style>
</head>
<body>
  <header>
    <h1>사진게시판</h1>
    <div class="controls">
      <button id="addBtn">사진 추가</button>
      <button id="clearBtn" class="small">모두 삭제</button>
    </div>
  </header>

  <main>
    <section class="gallery" id="gallery" aria-live="polite">
      <!-- 사진 카드가 여기로 추가됩니다 -->
    </section>
  </main>

  <footer>
    사진은 브라우저(localStorage)에 저장됩니다. 새로고침 후에도 남아있습니다. (용량 한계 있음)
  </footer>

  <!-- 추가 폼 모달 -->
  <div id="addModal" class="modal-backdrop hidden" role="dialog" aria-modal="true">
    <div class="modal">
      <h3 style="margin-top:0">사진 추가</h3>
      <div class="form-row">
        <label for="photoFile">사진 선택</label>
        <input id="photoFile" type="file" accept="image/*">
      </div>
      <div class="form-row">
        <label for="nickname">닉네임 (삭제 시 확인용 - 사진에는 표시되지 않습니다)</label>
        <input id="nickname" type="text" placeholder="게시자 닉네임을 입력하세요">
      </div>
      <p style="font-size:13px;color:#666;margin:0 0 8px 0">※ 닉네임은 내부적으로 사진과 함께 저장되지만 갤러리(사진 위)에는 표시되지 않습니다.</p>
      <div style="display:flex;gap:8px;align-items:center">
        <img id="previewSmall" src="" alt="미리보기" style="width:84px;height:64px;object-fit:cover;border-radius:6px;border:1px solid #eee;display:none">
        <div style="flex:1"></div>
      </div>
      <div class="btn-row">
        <button id="cancelAdd" class="small" style="background:#ddd;color:#111">취소</button>
        <button id="saveAdd">업로드</button>
      </div>
    </div>
  </div>

  <!-- 이미지 뷰어 모달 -->
  <div id="viewModal" class="modal-backdrop hidden" role="dialog" aria-modal="true">
    <div class="modal viewer" style="max-width:95%">
      <div style="display:flex;justify-content:flex-end"><button id="closeView" class="small">닫기</button></div>
      <img id="viewImg" src="" alt="전체 이미지">
      <p id="viewHint" style="color:#666;font-size:13px;margin-top:8px">사진에는 닉네임이 표시되지 않습니다.</p>
    </div>
  </div>

  <script>
    // 간단한 사진 게시판 (로컬스토리지 사용)
    const gallery = document.getElementById('gallery');
    const addBtn = document.getElementById('addBtn');
    const addModal = document.getElementById('addModal');
    const photoFile = document.getElementById('photoFile');
    const nickname = document.getElementById('nickname');
    const cancelAdd = document.getElementById('cancelAdd');
    const saveAdd = document.getElementById('saveAdd');
    const previewSmall = document.getElementById('previewSmall');
    const viewModal = document.getElementById('viewModal');
    const viewImg = document.getElementById('viewImg');
    const closeView = document.getElementById('closeView');
    const clearBtn = document.getElementById('clearBtn');

    const STORAGE_KEY = 'simple_photo_board_v1';

    // 사진 데이터 불러오기 (배열 of {id, dataUrl, time, nick})
    function loadPhotos(){
      try{
        const raw = localStorage.getItem(STORAGE_KEY);
        return raw ? JSON.parse(raw) : [];
      }catch(e){
        console.error('loadPhotos error', e);
        return [];
      }
    }

    function savePhotos(arr){
      localStorage.setItem(STORAGE_KEY, JSON.stringify(arr));
    }

    function renderGallery(){
      gallery.innerHTML = '';
      const photos = loadPhotos();
      if(photos.length === 0){
        gallery.innerHTML = '<p style="color:#666">등록된 사진이 없습니다. "사진 추가"를 눌러 업로드하세요.</p>';
        return;
      }
      photos.slice().reverse().forEach(p => { // 최신이 위로
        const card = document.createElement('article');
        card.className = 'card';
        card.dataset.id = p.id;
        // 이미지
        const img = document.createElement('img');
        img.src = p.dataUrl;
        img.alt = '사용자 업로드 사진';
        img.loading = 'lazy';
        img.addEventListener('click', ()=> openViewer(p.dataUrl));
        card.appendChild(img);

        const meta = document.createElement('div');
        meta.className = 'meta';
        const time = document.createElement('div');
        time.className = 'time';
        const d = new Date(p.time);
        time.textContent = d.toLocaleString();
        meta.appendChild(time);

        const actions = document.createElement('div');
        actions.className = 'actions';
        // 삭제 버튼: 누르면 닉네임 확인 창이 뜨고 일치하면 삭제
        const del = document.createElement('button');
        del.className = 'small';
        del.textContent = '삭제';
        del.addEventListener('click', ()=>{ 
          // 경고: prompt 문자열 안에 실제 개행을 넣으면 구문 오류가 발생하므로 한 줄로 합칩니다.
          const input = prompt('사진 삭제를 위해 업로더 닉네임을 정확히 입력하세요 (대소문자 구분 없이 비교됩니다)');
          if(input === null) return; // 취소
          const entered = input.trim();
          const stored = (p.nick || '').trim();
          // 대소문자 구분 없이 비교 (현재는 case-insensitive)
          if(stored.toLowerCase() === entered.toLowerCase()){
            if(confirm('닉네임이 확인되었습니다. 이 사진을 삭제하시겠습니까?')) removePhoto(p.id);
          }else{
            alert('닉네임이 일치하지 않습니다. 삭제할 수 없습니다.');
          }
        });
        actions.appendChild(del);

        meta.appendChild(actions);
        card.appendChild(meta);
        gallery.appendChild(card);
      });
    }

    function openViewer(src){
      viewImg.src = src;
      viewModal.classList.remove('hidden');
    }
    function closeViewer(){
      viewImg.src = '';
      viewModal.classList.add('hidden');
    }

    function removePhoto(id){
      const arr = loadPhotos().filter(x=> x.id !== id);
      savePhotos(arr);
      renderGallery();
    }

    addBtn.addEventListener('click', ()=>{
      photoFile.value = '';
      nickname.value = '';
      previewSmall.style.display = 'none';
      addModal.classList.remove('hidden');
    });

    cancelAdd.addEventListener('click', ()=> addModal.classList.add('hidden'));
    closeView.addEventListener('click', closeViewer);

    // 파일 선택 시 미리보기
    photoFile.addEventListener('change', (e)=>{
      const f = e.target.files && e.target.files[0];
      if(!f) { previewSmall.style.display='none'; return; }
      const reader = new FileReader();
      reader.onload = ()=>{
        previewSmall.src = reader.result;
        previewSmall.style.display = '';
      };
      reader.readAsDataURL(f);
    });

    // 저장 (닉네임은 입력받지만 갤러리에는 표시하지 않음)
    saveAdd.addEventListener('click', ()=>{
      const f = photoFile.files && photoFile.files[0];
      if(!f){ alert('사진 파일을 선택하세요.'); return; }
      const nick = nickname.value.trim();
      if(nick.length === 0){ 
        if(!confirm('닉네임 없이 업로드하시겠습니까? (닉네임 없이 업로드하면 삭제 시 닉네임 입력란에 빈값을 입력해야 삭제됩니다)')) return; 
      }
      const reader = new FileReader();
      reader.onload = ()=>{
        const arr = loadPhotos();
        arr.push({ id: 'p'+Date.now()+Math.floor(Math.random()*9999), dataUrl: reader.result, time: Date.now(), nick: nick });
        savePhotos(arr);
        addModal.classList.add('hidden');
        renderGallery();
      };
      reader.readAsDataURL(f);
    });

    clearBtn.addEventListener('click', ()=>{
      if(confirm('모든 사진을 삭제합니다. 계속할까요?')){
        localStorage.removeItem(STORAGE_KEY);
        renderGallery();
      }
    });

    // 키보드 ESC로 모달 닫기
    document.addEventListener('keydown', (e)=>{
      if(e.key === 'Escape'){
        if(!addModal.classList.contains('hidden')) addModal.classList.add('hidden');
        if(!viewModal.classList.contains('hidden')) closeViewer();
      }
    });

    // 초기 렌더
    renderGallery();
  </script>
</body>
</html>
