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

  const latency = 300;

  const socket = io("http://localhost:3000", {
    transports: ["websocket", "polling"],
  });

  let isHost = false,
    current = undefined,
    run = 0,
    thisStream;

  let pc = {},
    candidates = [];

  const addTrackToVideo = (i) => {
    // console.log(remoteStream, remoteStream.id);
    let vid = findOrCreateVid(i);
    console.log(vid);
    vid.srcObject = pc[i].remoteStream;
  };

  const findOrCreateVid = (n) => {
    let vid = document.querySelector(`#remote-video-${n}`);
    if (!vid) {
      vid = document.createElement("video");
      vid.setAttribute("class", "video svelte-uisvfq");
      vid.setAttribute("id", `remote-video-${n}`);
      vid.controls = true;
      vid.autoplay = true;
      vid.muted = true;
      document.querySelector("main").append(vid);
    }
    return vid;
  };

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
    navigator.mediaDevices
      .getUserMedia({
        audio: true,
        video: false,
      })
      .then((stream) => {
        let video = document.querySelector("#self-video");
        video.srcObject = stream;
        stream.getTracks().forEach((track) => pc[id].addTrack(track, stream));
        console.log("Adding tracks to stream.", id);
        video.onloadedmetadata = function (e) {
          video.play();
        };
      })
      .catch((err) => {
        console.log(`Unable to use MediaStream API : ${err}`);
      });
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
      setTimeout(() => {
        pc[from].setRemoteDescription(data);
        setTimeout(() => {
          send("SEND::CANDIDATE", candidates);
          // console.log(pc[from]);
        }, latency);
      }, latency);
    }

    if (data.type === "offer") {
      console.log("Recieved Offer :: ", { from, data });
      pc[from] = new RTCPeerConnection({
        iceServers: [
          {
            urls,
          },
        ],
        iceCandidatePoolSize: 2,
      });

      pc[from].connectedTo = from;

      pc[from].remoteStream = new MediaStream();

      addTrackToPeerConnection(from);

      pc[from].onicecandidate = (e) => {
        if (e.candidate) {
          candidates.push(e.candidate);
        }
      };

      pc[from].ontrack = (e) => {
        console.log("track from host", e);
        pc[e.target.connectedTo].remoteStream.addTrack(e.track);
        addTrackToVideo(e.target.connectedTo);
      };

      pc[from].onconnectionstatechange = (event) => {
        // console.log(event);
        state = `iceConnection: ${event.target.iceConnectionState} | iceGathering: ${event.target.iceGatheringState} | connectionState: ${event.target.connectionState}`;
        switch (event.target.connectionState) {
          case "connected": {
            console.log("The connection has become fully connected");
            break;
          }
          case "disconnected": {
            let vid = document.querySelector(
              `#remote-video-${event.target.connectedTo}`
            );
            vid.remove();
            break;
          }
          case "failed": {
            console.log(
              "One or more transports has terminated unexpectedly or in an error"
            );
            let vid = document.querySelector(
              `#remote-video-${event.target.connectedTo}`
            );
            vid.remove();
            break;
          }
          case "closed": {
            console.log("The connection has been closed");
            break;
          }
        }
      };

      console.log(pc);
      setTimeout(() => {
        pc[from].setRemoteDescription(data);
        setTimeout(() => createAnswer(from), latency);
      }, latency);
    }
  });

  socket.on("CANDIDATE", ({ from, data, type }) => {
    console.log("SOCKET ON :: CANDIDATE", { data });
    data &&
      setTimeout(() => {
        data.forEach((e) => pc[from].addIceCandidate(e));
      }, latency);
    console.log(pc[from]);
    // if (type !== "recv") {
    //   send("RECV::CANDIDATE", candidates);
    //   console.log("RECV CANDIDATES SEND");
    // }
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

    pc[id].remoteStream = new MediaStream();

    pc[id].connectedTo = id;

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
      console.log("track from guest", e);
      pc[e.target.connectedTo].remoteStream.addTrack(e.track);
      addTrackToVideo(e.target.connectedTo);
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

          // console.log("Sending Offer");
          send("SDP", sdp);
        })
        .catch(console.log);
    };

    pc[id].onconnectionstatechange = (event) => {
      // console.log(event);
      state = `iceConnection: ${event.target.iceConnectionState} | iceGathering: ${event.target.iceGatheringState} | connectionState: ${event.target.connectionState}`;
      switch (event.target.connectionState) {
        case "connected": {
          console.log("The connection has become fully connected");
          break;
        }
        case "disconnected": {
          let vid = document.querySelector(
            `#remote-video-${event.target.connectedTo}`
          );
          vid.remove();
          break;
        }
        case "failed": {
          console.log(
            "One or more transports has terminated unexpectedly or in an error"
          );
          let vid = document.querySelector(
            `#remote-video-${event.target.connectedTo}`
          );
          vid.remove();
          break;
        }
        case "closed": {
          console.log("The connection has been closed");
          break;
        }
      }
    };

    console.log(pc);
  });
</script>

<main>
  <video class="video" id="self-video" autoplay controls muted>
    <track kind="captions" />
  </video>
  <!-- <video class="video" id="remote-video" autoplay controls muted>
    <track kind="captions" />
  </video> -->
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
