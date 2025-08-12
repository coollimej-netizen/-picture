<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<title>사진 게시판</title>
<style>
  body { font-family: Arial, sans-serif; background: #f7f7f8; text-align: center; }
  .gallery { display: flex; flex-wrap: wrap; justify-content: center; gap: 10px; margin-top: 20px; }
  .gallery img { max-width: 200px; border-radius: 8px; }
</style>
</head>
<body>

<h1>사진 게시판</h1>
<input type="text" id="nickname" placeholder="닉네임 입력" />
<input type="file" id="photo" accept="image/*" />
<button id="uploadBtn">업로드</button>

<div class="gallery" id="gallery"></div>

<script>
const uploadBtn = document.getElementById('uploadBtn');
const nicknameInput = document.getElementById('nickname');
const photoInput = document.getElementById('photo');
const gallery = document.getElementById('gallery');
let photos = [];

uploadBtn.addEventListener('click', () => {
    const nickname = nicknameInput.value.trim();
    const file = photoInput.files[0];

    if (!nickname) return alert('닉네임을 입력하세요.');
    if (!file) return alert('사진을 선택하세요.');

    const reader = new FileReader();
    reader.onload = function(e) {
        photos.push({ nickname: nickname.toLowerCase(), src: e.target.result });
        renderGallery();
        nicknameInput.value = '';
        photoInput.value = '';
    };
    reader.readAsDataURL(file);
});

function renderGallery() {
    gallery.innerHTML = '';
    photos.forEach((photo, index) => {
        const img = document.createElement('img');
        img.src = photo.src;
        img.title = "삭제하려면 클릭하세요";
        img.addEventListener('click', () => {
            const input = prompt('사진 삭제를 위해 업로더 닉네임을 입력하세요 (대소문자 구분 없음)');
            if (input && input.toLowerCase() === photo.nickname) {
                photos.splice(index, 1);
                renderGallery();
            } else {
                alert('닉네임이 일치하지 않습니다.');
            }
        });
        gallery.appendChild(img);
    });
}
</script>

</body>
</html>
