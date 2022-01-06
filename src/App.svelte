<script>
  import { io } from "socket.io-client";
  $: state = "";
  const urls = [
    "stun:stun.infra.net:3478",
    "stun:stun.crononauta.com:3478",
    "stun:stun.xtratelecom.es:3478",
    "stun:stun3.l.google.com:19305",
    "stun:stun.coffee-sen.com:3478",
    "stun:stun.futurasp.es:3478",
    "stun:stun.millenniumarts.org:3478",
    "stun:stun.uls.co.za:3478",
    "stun:stun.oncloud7.ch:3478",
    "stun:stun1.l.google.com:19302",
    "stun:stun.telbo.com:3478",
    "stun:stun.fitauto.ru:3478",
    "stun:stun.edwin-wiegele.at:3478",
    "stun:stun.marble.io:3478",
    "stun:stun.lineaencasa.com:3478",
    "stun:stun.jumblo.com:3478",
    "stun:stun4.3cx.com:3478",
  ];

  const socket = io("https://3d89-103-221-209-13.ngrok.io/", {
    transports: ["websocket", "polling"],
  });

  let isHost = false,
    current = undefined,
    run = 0,
    thisStream,
    remoteStream = new MediaStream();

  let pc = {},
    candidates = [];

  const addTrackToVideo = () => {
    run += 1;
    if (run === 2) {
      document.querySelector("#remote-video").srcObject = remoteStream;
      // console.log(remoteStream);
    }
  };

  // const createOffer = (n) => {};

  const createAnswer = (n) => {
    console.log("Creating Answer");
    pc[n]
      .createAnswer({
        offerToReceiveAudio: true,
        offerToReceiveVideo: true,
      })
      .then((sdp) => {
        pc[n].setLocalDescription(sdp);

        console.log("Sending Answer");
        send("SDP", sdp);
      })
      .catch(console.log);
  };

  socket.on("connection::ok", (socket, host) => {
    console.log(`HOST :: ${host} :: ${socket}`);
  });

  const send = (event, payload, to) => {
    socket.emit(event, { to: current, data: payload });
  };

  const addTrackToPeerConnection = (id) => {
    thisStream &&
      thisStream
        .getTracks()
        .forEach((track) => pc[id].addTrack(track, thisStream));
  };

  socket.on("SDP", ({ from, data }) => {
    console.log(
      `SOCKET ON :: SDP ${isHost ? "PEER" : "REMOTE PEER"} ${
        data.type
      } :: ${from}`
    );
    current = from;

    if (data.type === "answer") {
      console.log("Recieved Answer :: ", { from, data });
      // console.log("addTrackToPeerConnection", thisStream);
      pc[from].setRemoteDescription(data);
      setTimeout(() => {
        send("SEND::CANDIDATE", candidates);
        console.log(pc[from]);
      }, 300);
    }
    // else
    if (data.type === "offer") {
      console.log("Recieved Offer :: ", { from, data });
      // Create Peer Connection for recieving track. :: Guest
      pc[from] = new RTCPeerConnection({
        iceServers: [
          {
            urls,
          },
        ],
        iceCandidatePoolSize: 2,
      });

      addTrackToPeerConnection(from);

      state = pc[from].iceGatheringState;

      pc[from].onicecandidate = (e) => {
        if (e.candidate) {
          candidates.push(e.candidate);
        }
      };

      pc[from].oniceconnectionstatechange = (e) =>
        (state = e.currentTarget.connectionState);

      pc[from].ontrack = (e) => {
        // console.log(e);
        remoteStream.addTrack(e.track);
        addTrackToVideo();
      };

      // console.log(pc);
      pc[from].setRemoteDescription(data);
      setTimeout(() => {
        // console.log("addTrackToPeerConnection", thisStream);
        setTimeout(() => createAnswer(from), 300);
      }, 300);
    }
  });

  socket.on("CANDIDATE", ({ from, data, type }) => {
    console.log("SOCKET ON :: CANDIDATE", { data });
    data && data.forEach((e) => pc[from].addIceCandidate(e));
    console.log(pc[from]);
    type !== "recv" && send("RECV::CANDIDATE", candidates);
  });

  socket.on("is::new", (id) => {
    current = id;
    pc[id] = new RTCPeerConnection({
      iceServers: [
        {
          urls,
        },
      ],
      iceCandidatePoolSize: 2,
    });

    addTrackToPeerConnection(id);

    state = pc[id].iceGatheringState;

    pc[id].onicecandidate = (e) => {
      if (e.candidate) {
        candidates.push(e.candidate);
      }
    };

    pc[id].oniceconnectionstatechange = (e) =>
      (state = e.currentTarget.connectionState);

    pc[id].ontrack = (e) => {
      // console.log(e);
      remoteStream.addTrack(e.track);
      addTrackToVideo();
    };

    // createOffer(id);
    pc[id].onnegotiationneeded = () => {
      console.log("Creating Offer");
      pc[id]
        .createOffer({
          offerToReceiveAudio: true,
          offerToReceiveVideo: true,
        })
        .then((sdp) => {
          pc[id].setLocalDescription(sdp);

          console.log("Sending Offer");
          send("SDP", sdp);
        })
        .catch(console.log);
    };
    // console.log(pc);
  });

  let constraints = {
    audio: true,
    video: {
      width: { min: 1024, ideal: 1280, max: 1920 },
      height: { min: 576, ideal: 720, max: 1080 },
    },
  };

  navigator.mediaDevices
    .getUserMedia(constraints)
    .then(function (stream) {
      /* use the stream */
      let video = document.querySelector("#self-video");
      video.srcObject = stream;
      thisStream = stream;
      video.onloadedmetadata = function (e) {
        video.play();
      };
    })
    .catch(function (err) {
      /* handle the error */
      console.log(`Unable to use MediaStream API : ${err}`);
    });
</script>

<main>
  <video class="video" id="self-video" autoplay controls muted>
    <track kind="captions" />
  </video>
  <video class="video" id="remote-video" autoplay controls muted>
    <track kind="captions" />
  </video>
  <button class="states">{state}</button>
</main>

<style>
  main {
    width: 100%;
    height: 100%;
    margin: auto;
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: space-evenly;
    background-color: #000;
    border-radius: 1rem;
  }
  .video {
    aspect-ratio: 16/9;
    background-color: transparent;
    width: 480px;
    border: 4px solid #3f3f3f;
    border-radius: 15px;
    cursor: pointer;
  }

  .states {
    position: absolute;
    bottom: 1rem;
    right: 1rem;
    padding: 0.5rem 1rem;
    text-transform: capitalize;
    border-radius: 0.5rem;
  }

  @media only screen and (min-width: 600px) {
    .video {
      aspect-ratio: 4/3;
      background-color: transparent;
      width: 240px;
    }
  }
</style>
